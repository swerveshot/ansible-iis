# Ansible IIS Deployment Project

[![CI](https://github.com/swerveshot/ansible-iis/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/swerveshot/ansible-iis/actions/workflows/ci.yml)

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
| pypsrp       | 0.7.0+                          |

### Target server (Windows)

| Component      | Requirement                          |
|----------------|--------------------------------------|
| OS             | Windows Server 2022                  |
| PowerShell     | 5.1+                                 |
| .NET Framework | 4.7.2+ (for PSRemoting)              |
| PSRemoting     | Enabled (see below)                  |
| Disk layout    | C:\ (OS), D:\ (data)                 |
| Firewall       | TCP 5986 open from control node      |

### Configure PSRemoting on the target server

Ansible connects via **PSRP** (PowerShell Remoting Protocol), which uses the same
WinRM transport ports (5985/5986) but offers lower per-task overhead through
connection reuse and pipelining.

Run the following PowerShell script as Administrator on each target server:

```powershell
# Enable PowerShell Remoting (required for PSRP)
Enable-PSRemoting -Force -SkipNetworkProfileCheck

# Create a self-signed certificate for HTTPS (for non-production environments)
$cert = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME -CertStoreLocation Cert:\LocalMachine\My
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $cert.Thumbprint -Force

# Open firewall for WinRM HTTPS
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -LocalPort 5986 -Protocol TCP -Action Allow

# Start WinRM service
Start-Service WinRM
Set-Service WinRM -StartupType Automatic
```

> **Production recommendation:** Use a CA-signed certificate and NTLM or Kerberos
> authentication. Set `ansible_psrp_cert_validation: validate` in the production
> group_vars.

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
├── ci/
│   └── inventory/                       # CI-specific inventory (GitHub Actions)
│       ├── hosts.yml
│       └── group_vars/all.yml           # CI overrides (C-drive, .NET 8 only, iana_timezone)
├── inventory/
│   ├── hosts.yml                        # OTAP inventory
│   └── group_vars/
│       ├── all/main.yml                 # Global variables
│       ├── windows/main.yml             # PSRP connection settings
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
            ├── iis_configure.yml        # IIS configuration and firewall verification
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

**.NET download URLs and checksums** are fetched automatically at runtime from the
[Microsoft .NET Releases API](https://dotnetcli.blob.core.windows.net/dotnet/release-metadata/releases-index.json).
No manual updates are needed when a new patch release is published. Only the
`dotnet_channel_version` entries in `roles/iis/defaults/main.yml` need updating
when a new major version is added or a version reaches end-of-life.

**URL Rewrite Module** URL is hardcoded in `roles/iis/defaults/main.yml`; the
product has been frozen since 2018 and the URL is stable.

Check supported .NET versions at: https://dotnet.microsoft.com/en-us/platform/support/policy

## CI/CD

A GitHub Actions pipeline runs on every push:

| Job | Runner | Trigger |
|---|---|---|
| Lint (`ansible-lint`) | ubuntu-latest | every push |
| Syntax check | ubuntu-latest | every push |
| Integration test + idempotency | windows-2022 | after lint + syntax |

The integration job installs IIS on the runner itself using WSL as the Ansible
control node, then verifies IIS is listening and serving requests.
