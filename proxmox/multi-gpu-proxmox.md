# Multi-GPU Proxmox Configuration

This setup uses Proxmox to contribute multiple GPUs to the Lilypad Network. 

**Cloud Considerations:**
When operating in a cloud environment, opt for a bare metal instance (denoted as `.metal`) or an instance that supports nested virtualization. We do not recommend this as a resource provider; renting gpu’s from cloud providers can be very expensive. 

By using Proxmox we create individual virtual machines (VMs) for each GPU. Each VM runs a separate instance of Lilypad, enabling GPU usage and isolation.

**Setup Overview:** To setup each gpu with an instance of Lilypad, this includes: Configure IOMMU within the bios for GPU passthrough, create a base VM template, and clone the VM for each available GPU. Each GPU operates in its dedicated environment. This guide is designed to be provider and server agnostic, assuming that you have network access to the devices.

You need a public/private key on each VM. There are many ways to create and fund your wallets. I chose to create four separate wallets for each of the GPU’s in my setup and funded them manually. **Disclaimer**: be careful with scripting these operations!

Steps: 

1. **Proxmox Configuration**:
    - Ensure Proxmox is installed and configured on your server(I used Proxmox 8.1).
    - Configure GPU passthrough on Proxmox. This involves enabling IOMMU in your BIOS and making necessary changes to your Proxmox configuration files. More information here: [https://www.linux-kvm.org/page/How_to_assign_devices_with_VT-d_in_KVM](https://www.linux-kvm.org/page/How_to_assign_devices_with_VT-d_in_KVM)
2. **Prepare a VM Template**:
    - Create a base VM template in Proxmox with the necessary configurations (Ubuntu 22.04, lilypad, wallet+funds).
    - Ensure the template has no GPU assigned initially.
3. **Clone the VM with** **Ansible Playbook**:
    - Write an Ansible playbook to automate the creation of VMs, assignment of GPUs, and setup of Lilypad instances. (base this playbook on the number of GPU’s you have) One VM for every GPU.

Here is a detailed breakdown:

### 1. Proxmox Configuration

**Enable IOMMU:**

1. Edit `/etc/default/grub` to include:
    
    ```bash
    GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
    ```
    
    For AMD processors, use `amd_iommu=on`.
    
2. Update GRUB and reboot:
    
    ```bash
    update-grub
    reboot
    
    ```
    

**Edit Proxmox Configuration:**

1. Edit `/etc/modules` to include:
    
    ```bash
    vfio
    vfio_iommu_type1
    vfio_pci
    vfio_virqfd
    
    ```
    
2. Update initramfs:
    
    ```bash
    update-initramfs -u
    
    ```
    

### 2. Prepare a VM Template

1. Create a base VM in Proxmox with the desired operating system. I used Ubuntu 22.04 but pending on your requirements, using a lightweight OS might be a better option.
2. Install any necessary dependencies and software. Lilypad, Bacalhau, Nvidia Container Toolkit, Nvidia Drivers!
3. Convert the VM to a template.

### 3. Ansible Playbook

**Ansible Inventory File (`hosts.ini`):**

```
[proxmox]
proxmox_host ansible_host=your_proxmox_ip ansible_user=root ansible_password=your_proxmox_password

```

**Ansible Playbook (`setup_vms.yml`):**

```yaml
- name: Setup VMs for GPUs
  hosts: proxmox
  tasks:
    - name: Get list of available GPUs
      shell: lspci | grep -i nvidia
      register: gpu_list

    - name: Create VMs for each GPU
      loop: "{{ gpu_list.stdout_lines }}"
      vars:
        vm_id: "{{ 100 + loop.index }}"
        vm_name: "gpu-vm-{{ loop.index }}"
      block:
        - name: Create VM from template
          shell: |
            qm clone 9000 {{ vm_id }} --name {{ vm_name }}
            qm set {{ vm_id }} --memory 4096 --cores 4 --net0 virtio,bridge=vmbr0
            qm set {{ vm_id }} --hostpci0 0000:{{ item.split()[0] }}

        - name: Start the VM
          shell: qm start {{ vm_id }}

        - name: Wait for VM to be ready
          wait_for:
            host: "{{ vm_name }}"
            port: 22
            delay: 10
            timeout: 300

        - name: Install Docker, Nvidia Container Toolkit, Bacalhau, and Lilypad
          shell: |
            ssh root@{{ vm_name }} << EOF
              # Install Docker
              apt-get update
              apt-get install -y apt-transport-https ca-certificates curl software-properties-common
              curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
              add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
              apt-get update
              apt-get install -y docker-ce docker-ce-cli containerd.io

              # Install Nvidia Container Toolkit
              distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
              curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | apt-key add -
              curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
              apt-get update
              apt-get install -y nvidia-container-toolkit
              nvidia-ctk runtime configure --runtime=docker
              systemctl restart docker

              # Install Bacalhau
              cd /tmp
              wget https://github.com/bacalhau-project/bacalhau/releases/download/v1.3.2/bacalhau_v1.3.2_linux_amd64.tar.gz
              tar xfv bacalhau_v1.3.2_linux_amd64.tar.gz
              mv bacalhau /usr/bin/bacalhau
              mkdir -p /app/data/ipfs
              chown -R root /app/data

              # Install Lilypad
              OSARCH=$(uname -m | awk '{if ($0 ~ /arm64|aarch64/) print "arm64"; else if ($0 ~ /x86_64|amd64/) print "amd64"; else print "unsupported_arch"}')
              OSNAME=$(uname -s | awk '{if ($1 == "Darwin") print "darwin"; else if ($1 == "Linux") print "linux"; else print "unsupported_os"}')
              curl https://api.github.com/repos/lilypad-tech/lilypad/releases/latest | grep "browser_download_url.*lilypad-$OSNAME-$OSARCH-gpu" | cut -d : -f 2,3 | tr -d \" | wget -qi - -O lilypad
              chmod +x lilypad
              mv lilypad /usr/local/bin/lilypad

              # Create environment file
              mkdir -p /app/lilypad
              echo "WEB3_PRIVATE_KEY=<YOUR_PRIVATE_KEY>" > /app/lilypad/resource-provider-gpu.env

              # Create systemd unit for Bacalhau
              cat << EOT > /etc/systemd/system/bacalhau.service
              [Unit]
              Description=Lilypad V2 Bacalhau
              After=network-online.target
              Wants=network-online.target systemd-networkd-wait-online.service

              [Service]
              Environment="LOG_TYPE=json"
              Environment="LOG_LEVEL=debug"
              Environment="HOME=/app/lilypad"
              Environment="BACALHAU_SERVE_IPFS_PATH=/app/data/ipfs"
              Restart=always
              RestartSec=5s
              ExecStart=/usr/bin/bacalhau serve --node-type compute,requester --peer none --private-internal-ipfs=false

              [Install]
              WantedBy=multi-user.target
              EOT

              # Create systemd unit for GPU provider
              cat << EOT > /etc/systemd/system/lilypad-resource-provider.service
              [Unit]
              Description=Lilypad V2 Resource Provider GPU
              After=network-online.target
              Wants=network-online.target systemd-networkd-wait-online.service

              [Service]
              Environment="LOG_TYPE=json"
              Environment="LOG_LEVEL=debug"
              Environment="HOME=/app/lilypad"
              Environment="OFFER_GPU=1"
              EnvironmentFile=/app/lilypad/resource-provider-gpu.env
              Restart=always
              RestartSec=5s
              ExecStart=/usr/local/bin/lilypad resource-provider 

              [Install]
              WantedBy=multi-user.target
              EOT

              # Reload systemd and start services
              systemctl daemon-reload
              systemctl enable bacalhau
              systemctl enable lilypad-resource-provider
              systemctl start bacalhau
              systemctl start lilypad-resource-provider
            EOF

```

**Steps in the process:**

1. **Inventory File:** Defines the Proxmox host.
2. **Playbook:**
    - **Get list of GPUs:** Uses `lspci` to find all NVIDIA GPUs.
    - **Create VMs:** For each GPU, it clones the base template, assigns a unique VM ID and name, allocates resources, and attaches a GPU.
    - **Start the VM:** Boots up the VM.
    - **Install Lilypad:** Runs a script to install and configure Lilypad on each VM.

### Running the Playbook

Ensure you have Ansible installed on your control machine. Run the playbook with:

```bash
ansible-playbook -i hosts.ini setup_vms.yml

```
