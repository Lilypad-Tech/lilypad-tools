# Lilypad Update Script

This script automatically updates your Lilypad binary to the latest version.

## Usage

1. Download the script:
    ```
    curl -O https://raw.githubusercontent.com/lilypad-tech/lilypad-tools/main/update/update_lilypad.sh
    ```

2. Make the script executable:
    ```
    chmod +x update_lilypad.sh
    ```

3. Run the script with sudo:
    ```
    sudo ./update_lilypad.sh
    ```

The script will check for updates, download the latest version if available, and restart the Lilypad service.

Note: Ensure you have curl installed and your system meets Lilypad's requirements.
