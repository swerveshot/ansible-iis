# Ansible IIS Deployment Project

Ansible project for automated installation and configuration of IIS on Windows Server 2022.

## Features

- IIS installation with full feature set
- IIS URL Rewrite Module 2.1
- .NET 8 (LTS), .NET 9 (STS) and .NET 10 (LTS) Hosting Bundles (x64 and x86)
- Data segregation: OS on C:\, all IIS data on D:\
- OTAP inventory (Development, Test, Acceptance, Production)
- Selective execution via Ansible tags

## Requirements

### Management workstation (Ansible control node)

| Component    | Minimum version                 |
|--------------|---------------------------------|
| OS           | Linux (Ubuntu 22.04+ / RHEL 9+) |
| Python       | 3.11+                           |
| ansible-core | 2.16+                           |
| pywinrm      | 0.4.3+                          |

### Target server (Windows)

| Component      | Requirement                          |
|----------------|--------------------------------------|
| OS             | Windows Server 2022                  |
| PowerShell     | 5.1+                                 |
| .NET Framework | 4.7.2+ (for WinRM)                   |
| WinRM          | Configured (see below)               |
| Disk layout    | C:\ (OS), D:\ (data)                 |
| Firewall       | TCP 5986 open from control node      |

### Configure WinRM on the target server

Run the following PowerShell script as Administrator on each target server:

```powershell
# Configure WinRM with HTTPS (recommended for production)
winrm quickconfig -q
winrm set winrm/config/service/auth '@{Basic="false"; Kerberos="true"; Negotiate="true"}'

# Create a self-signed certificate for WinRM HTTPS (for test environments)
$cert = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME -CertStoreLocation Cert:\LocalMachine\My
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $cert.Thumbprint -Force

# Open firewall for WinRM HTTPS
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -LocalPort 5986 -Protocol TCP -Action Allow

# Start WinRM service
Start-Service WinRM
Set-Service WinRM -StartupType Automatic
```

> **Production recommendation:** Use a CA-signed certificate and Kerberos or CredSSP authentication.

## Installation

### 1. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 2. Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure inventory

Edit `inventory/hosts.yml` and the corresponding `group_vars` with the correct hostnames and IP addresses.

Store passwords in **Ansible Vault**:

```bash
ansible-vault encrypt_string 'SuperSecretPassword123!' --name 'ansible_password'
```

Add the output to the relevant `group_vars` file.

## Usage

### Full installation

```bash
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --ask-vault-pass
```

### Per environment

```bash
# Development only
ansible-playbook playbooks/site.yml -i inventory/hosts.yml -l otap_o

# Production only
ansible-playbook playbooks/site.yml -i inventory/hosts.yml -l otap_p
```

### Selective execution with tags

```bash
# OS requirements only
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags os_requirements

# IIS features only
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags iis_features

# IIS configuration only (paths, logging, etc.)
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags iis_configure

# Install IIS Rewrite Module
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags iis_rewrite

# Install .NET runtimes
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags dotnet

# Specific .NET version
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags dotnet_8
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags dotnet_9
ansible-playbook playbooks/site.yml -i inventory/hosts.yml --tags dotnet_10
```

## Directory structure

```
iis-ansible/
├── README.md
├── ansible.cfg                          # Ansible configuration
├── requirements.txt                     # Python dependencies
├── requirements.yml                     # Ansible collection dependencies
├── inventory/
│   ├── hosts.yml                        # OTAP inventory
│   └── group_vars/
│       ├── all/main.yml                 # Global variables
│       ├── windows/main.yml             # WinRM connection settings
│       ├── otap_o/main.yml              # Development-specific variables
│       ├── otap_t/main.yml              # Test-specific variables
│       ├── otap_a/main.yml              # Acceptance-specific variables
│       └── otap_p/main.yml              # Production-specific variables
├── playbooks/
│   └── site.yml                         # Main playbook
└── roles/
    └── iis/
        ├── defaults/main.yml            # Default role values
        ├── vars/main.yml                # Internal role variables
        ├── handlers/main.yml            # Handlers (service restarts)
        ├── meta/main.yml                # Role metadata and dependencies
        └── tasks/
            ├── main.yml                 # Task orchestration
            ├── os_requirements.yml      # OS preparation
            ├── iis_features.yml         # IIS feature installation
            ├── iis_configure.yml        # IIS configuration (D-drive)
            ├── iis_rewrite.yml          # URL Rewrite module
            └── _dotnet_version_install.yml  # .NET runtime helper (single version)
```

## .NET version lifecycle (as of May 2026)

| Version  | Type | Release date | EOL date     | Status        |
|----------|------|--------------|--------------|---------------|
| .NET 8   | LTS  | Nov 2022     | Nov 10, 2026 | Supported     |
| .NET 9   | STS  | Nov 2024     | Nov 10, 2026 | Supported     |
| .NET 10  | LTS  | Nov 2025     | Nov 14, 2028 | Supported     |

## Maintenance

Download URLs for .NET and the URL Rewrite Module are configured as variables in `roles/iis/defaults/main.yml` (`iis_dotnet_versions`) and `roles/iis/vars/main.yml`.
Update these variables when a new patch level becomes available.

Check supported versions at: https://dotnet.microsoft.com/en-us/platform/support/policy
