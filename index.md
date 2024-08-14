Welcome to the Terraform and Ansible labs.

These are hands-on resources to help you learn Terraform and Ansible.

## Pre-reqs

Ensure you have the following installed on your local machine:

- Docker
- Terraform
- Ansible
- kubectl
- Node.js and npm

## Setting up Terraform

### Terraform on Windows

1. Download the Terraform ZIP file from the [official website](https://www.terraform.io/downloads.html).
2. Extract the ZIP file to a directory, e.g., `C:\terraform`.
3. Add the directory to your system PATH:
   - Right-click on 'This PC' and choose 'Properties'.
   - Click on 'Advanced system settings'.
   - Click on 'Environment Variables'.
   - Under 'System variables', find 'Path', click 'Edit'.
   - Click 'New' and add the path to your Terraform directory.
4. Open a new command prompt and verify the installation:
   ```
   terraform version
   ```

### Terraform on Linux

1. Update package index and install required packages:
   ```bash
   sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
   ```
2. Add the HashiCorp GPG key:
   ```bash
   curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
   ```
3. Add the official HashiCorp repository:
   ```bash
   sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
   ```
4. Update and install Terraform:
   ```bash
   sudo apt-get update && sudo apt-get install terraform
   ```
5. Verify the installation:
   ```bash
   terraform version
   ```

### Terraform on macOS

1. Install Homebrew if not already installed:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
2. Install Terraform using Homebrew:
   ```bash
   brew install terraform
   ```
3. Verify the installation:
   ```bash
   terraform version
   ```

## Setting up Ansible

### Ansible on Windows

Ansible doesn't run natively on Windows. The recommended approach is to use Windows Subsystem for Linux (WSL).

1. Enable WSL by opening PowerShell as Administrator and running:
   ```powershell
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   ```
2. Restart your computer.
3. Install a Linux distribution from the Microsoft Store (e.g., Ubuntu).
4. Launch the Linux distribution and follow the Linux instructions below.

### Ansible on Linux

1. Update package index and install Ansible:
   ```bash
   sudo apt update
   sudo apt install ansible
   ```
2. Verify the installation:
   ```bash
   ansible --version
   ```

### Ansible on macOS

1. Install Ansible using Homebrew:
   ```bash
   brew install ansible
   ```
2. Verify the installation:
   ```bash
   ansible --version
   ```

## Part 1. Terraform only

- [Local K3s Tutorial with Terraform](labs/terraform-only/README.md)

## Part 2. Terraform and Ansible

- [Local K3s Tutorial with Terraform and Ansible](labs/terraform-ansible/README.md)

## Part 3. Exercises

- [K3s and Terraform Exercises](exercises/terraform-only/README.md)
- [K3s Terraform and Ansible Exercises](exercises/terraform-ansible/README.md)
