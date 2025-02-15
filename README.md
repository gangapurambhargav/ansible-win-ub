# ansible-win-ub
# Setting Up Ansible on Ubuntu to Manage Windows Machines

## Part 1: Understanding Ansible Components

### Control Node

- This is the machine where Ansible is installed.
- It connects to and manages other machines (managed nodes) via SSH or WinRM.

### Managed Node

- These are the machines that Ansible controls.
- They can run different operating systems like Windows, CentOS, or Debian.

### Why Use Ansible?

- **Configuration Management**: Automate system configurations.
- **Provisioning**: Set up servers and applications efficiently.
- **Deployment**: Deploy applications at scale.
- **Network Automation**: Manage network devices and settings.

---

## Part 2: Installing Ansible on Ubuntu

### Update Your System

```bash
sudo apt update
sudo apt upgrade -y
```

### Install Ansible

```bash
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

### Verify Installation

```bash
ansible --version
```

---

## Part 3: Setting Up Passwordless Authentication

To connect to managed nodes, we need passwordless SSH authentication:

### Using Public Key

```bash
ssh-copy-id -f "-o IdentityFile <PATH TO PEM FILE>" ubuntu@<INSTANCE-PUBLIC-IP>
```

- **ssh-copy-id**: Copies your public key to the remote machine.
- **-f**: Forces key copying if keys already exist.
- **-o IdentityFile**: Specifies the private key file.
- **ubuntu@**: Connects as the Ubuntu user.

### Using PasswordAuth in SSHD

If passwordless authentication isn't possible, enable password-based SSH:

1. Edit SSH config file:

```bash
sudo nano /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
```

2. Change `PasswordAuthentication` to `yes`.
3. Restart SSH:

```bash
sudo systemctl restart ssh
```

---

## Part 4: Configuring Windows for Management

### Set Execution Policy and Enable WinRM

Open PowerShell as Administrator and run:

```powershell
Set-ExecutionPolicy RemoteSigned -Force
Enable-PSRemoting -Force
```

### Configure WinRM

```powershell
winrm quickconfig
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
```

### Open Firewall Ports

```powershell
New-NetFirewallRule -DisplayName "WinRM 5985" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5985
```

---

## Part 5: Configuring Ansible Inventory

### Edit the Hosts File

```bash
sudo nano /etc/ansible/hosts
```

#### Inventory File Example hosts.ini

```ini
[windows]
windows1 ansible_host=192.168.X.X
windows2 ansible_host=192.168.X.X

[windows:vars]
ansible_user=Administrator
ansible_password=YourPassword
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
```

---

## Part 6: Testing Connectivity

### Ping All Managed Nodes

```bash
ansible -i inventory.ini -m ping all
```

### Ping Only Windows Hosts

```bash
ansible windows -m win_ping
```

### Run Commands on Managed Nodes

Install OpenJDK on all nodes:

```bash
ansible -i inventory.ini -m shell -a 'apt install openjdk' all
```

### Grouping in Hosts.ini

```ini
[app]
192.168.200.21

[db]
192.168.200.22
```



### Ping Specific Groups

```bash
ansible -i inventory.ini -m ping app
ansible -i inventory.ini -m ping db
```

---

## Part 7: Running Ansible Commands

### Two Ways to Run Commands

1. **Playbooks** - Automate multiple tasks at once.
2. **Ad-hoc Commands** - Execute simple one-line commands.

### Example Ad-hoc Commands

Check Windows connectivity:

```bash
ansible -i inventory.ini -m win_ping windows1
```

Run multiple commands for system info:

```bash
ansible -i inventory.ini -m shell -a 'whoami' all
```

---

## Part 8: Windows System Information Playbook

```yaml
---
- hosts: windows
  gather_facts: no

  tasks:
    - block:
        - name: User and System Information
          win_command: whoami
          register: whoami_output
        - debug:
            msg: "The current user is {{ whoami_output.stdout }}"

        - name: System Information
          win_command: systeminfo
          register: systeminfo_output
        - debug:
            msg: "System information: {{ systeminfo_output.stdout }}"
      tags: ["info", "system"]

    - block:
        - name: Network Information
          win_command: ipconfig /all
          register: ipconfig_output
        - debug:
            msg: "Full IPCONFIG output: {{ ipconfig_output.stdout }}"
```
