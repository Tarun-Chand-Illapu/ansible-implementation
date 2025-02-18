# Ansible Automation Setup in Azure Tenant (Windows & Linux) - Complete Guide

## 📌 Overview
This guide provides a step-by-step approach to setting up Ansible automation for Windows and Linux servers in an Azure tenant. It covers everything from creating Azure resources to configuring the Ansible control node and target machines and troubleshooting errors faced during setup.

## 🔹 1. Setting Up Azure Resources

### 1.1 Create Resource Group
```bash
az group create --name Ansible-RG --location eastus
```

### 1.2 Create Virtual Network & Subnet
```bash
az network vnet create --resource-group Ansible-RG --name ansible-vnet \
    --address-prefix 10.0.0.0/16 --subnet-name ansible-subnet --subnet-prefix 10.0.1.0/24
```

### 1.3 Create Ansible Control Node (Linux VM)
```bash
az vm create --resource-group Ansible-RG --name ansible-control \
    --image UbuntuLTS --vnet-name ansible-vnet --subnet ansible-subnet \
    --admin-username azureuser --generate-ssh-keys --size Standard_B2s
```

### 1.4 Create Windows & Linux Target Machines
```bash
# Linux Target
az vm create --resource-group Ansible-RG --name ansible-target-1 \
    --image UbuntuLTS --vnet-name ansible-vnet --subnet ansible-subnet \
    --admin-username azureuser --size Standard_B2s

# Windows Target
az vm create --resource-group Ansible-RG --name win-target \
    --image Win2022Datacenter --vnet-name ansible-vnet --subnet ansible-subnet \
    --admin-username azureadmin --admin-password "Secure@1234" --size Standard_B2s
```

## 🔹 2. Configuring Ansible Control Node

### 2.1 Install Ansible & Dependencies
```bash
sudo apt update && sudo apt install -y ansible python3-pip
```

### 2.2 Create SSH Keys & Copy to Linux Targets
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
ssh-copy-id azureuser@ansible-target-1
```

### 2.3 Install Python Virtual Environment
```bash
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install ansible
```

## 🔹 3. Configuring Windows Target for Ansible (WinRM Setup)

### 3.1 Enable WinRM & Create Firewall Rule
Run the following on the Windows target:
```powershell
Enable-PSRemoting -Force
winrm quickconfig -Force
New-NetFirewallRule -DisplayName "Allow WinRM" -Direction Inbound -Protocol TCP -LocalPort 5985 -Action Allow
```

### 3.2 Set Up Ansible Authentication
Edit `/etc/ansible/hosts`:
```ini
[linux_servers]
ansible-target-1 ansible_host=10.0.1.10 ansible_user=azureuser ansible_python_interpreter=/usr/bin/python3

[windows_servers]
win-target ansible_host=10.0.1.20 ansible_user=azureadmin ansible_password=Secure@1234 \
    ansible_port=5985 ansible_connection=winrm ansible_winrm_transport=basic \
    ansible_shell_type=powershell ansible_python_interpreter=auto_silent
```

## 🔹 4. Running Ansible Tests

### 4.1 Verify Connectivity
```bash
ansible all -m ping
```
✅ **Expected Output:**
```json
win-target | SUCCESS => {
    "ping": "pong"
}
ansible-target-1 | SUCCESS => {
    "ping": "pong"
}
```

## 🔹 5. Running Ansible Playbook for Updates & Health Checks

### 5.1 Create Playbook (`update-patch-all.yml`)
```yaml
- name: Apply Security Updates & Perform Health Check on Linux & Windows
  hosts: all
  tasks:
    - name: Install security updates (Linux)
      apt:
        update_cache: yes
        upgrade: dist
      when: ansible_os_family == "Debian"

    - name: Install Windows Updates (Security & Definition Updates)
      win_updates:
        category_names:
          - SecurityUpdates
          - DefinitionUpdates
      when: ansible_os_family == "Windows"

    - name: Check Uptime (Linux)
      command: uptime
      register: uptime_output
      when: ansible_os_family == "Debian"

    - name: Check Uptime (Windows)
      win_shell: systeminfo | find "System Boot Time"
      register: windows_uptime
      when: ansible_os_family == "Windows"
```

### 5.2 Run the Playbook
```bash
ansible-playbook update-patch-all.yml
```

✅ **Expected Output:**
```json
PLAY RECAP
ansible-target-1 : ok=11  changed=7  failed=0  skipped=10  rescued=0  ignored=0
win-target      : ok=9   changed=7  failed=0  skipped=12  rescued=0  ignored=0
```

## 🔹 6. Viewing Logs & Verifying Updates

### 6.1 Check Ansible Logs
```bash
ls -lah /var/log/ansible_health_checks_*
```

### 6.2 View Windows Update Logs
```powershell
Get-WindowsUpdateLog
```

## 🚀 Conclusion
This guide provides a complete step-by-step process to configure Ansible in an Azure environment, set up Linux & Windows targets, execute security patches, and perform system health checks.
