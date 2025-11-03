# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible playbook designed to automatically optimize and secure Ubuntu servers for Kamal deployment. The project consists of modular roles that handle different aspects of server provisioning and security hardening.

## Key Commands

Install Ansible dependencies:
```bash
ansible-galaxy install -r requirements.yml
```

Run the main provisioning playbook:
```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini playbook.yml
```

Note: Add `--ask-pass` flag if you need to provide the SSH password interactively (when SSH keys aren't set up yet).

Run Scaleway provisioning (creates instances then provisions them):
```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook scaleway.yml
```

## Architecture

### Main Playbooks
- `playbook.yml` - Main server provisioning playbook targeting `webservers` group
- `scaleway.yml` - Creates Scaleway compute instances then imports the main playbook

### Role Execution Order
The main playbook executes roles in this specific order:
1. `wait_for_connection` - Ensures SSH connectivity before proceeding
2. `packages` - Updates system packages and installs essential tools (including auditd)
3. `sysctl` - Applies kernel hardening parameters for network and system security
4. `docker` - Installs and configures Docker with security daemon settings
5. `deployer` - Creates deployer user with SSH keys for Kamal deployments
6. `firewall` - Configures UFW firewall rules
7. `security` - Configures SSH hardening, fail2ban jails, and unattended upgrades
8. `auditd` - Sets up comprehensive system auditing for security monitoring
9. `geerlingguy.swap` - External role for swap file configuration
10. `reboot_if_needed` - Reboots server if kernel updates require it

### Configuration Files
- `hosts.ini` - Ansible inventory defining target servers in `webservers` group (copy from `hosts.ini.example` to get started)
- `requirements.yml` - External Ansible collections and roles dependencies
- Variables are configured directly in `playbook.yml` under the `vars` section
- `roles/scaleway/vars/main.yml` - Scaleway configuration (copy from `main.yml.example` and configure API credentials)

### Key Features
- **Idempotent operations**: Roles check for existing configurations before making changes
- **Security hardening**:
  - Removes snap packages
  - Configures fail2ban with SSH protection (3600s ban after 5 failed attempts)
  - UFW firewall with rate limiting on SSH
  - Disables SSH password login, adds MaxAuthTries limit
  - Kernel hardening via sysctl (protection against SYN floods, IP spoofing, etc.)
  - Docker daemon security configuration (log rotation, disabled ICC, no-new-privileges)
  - Comprehensive auditd rules monitoring sudo, SSH, Docker, user changes, and system calls
- **Kamal preparation**: Creates dedicated deployer user with SSH key management
- **Optional cloud provisioning**: Scaleway integration for automated server creation

### SSH Key Management
The deployer role generates ed25519 SSH keys and stores private keys locally in `./keys/{{ inventory_hostname }}/deployer_id_ed25519` for use with Kamal deployments.

### Variable Configuration
Override default settings in `playbook.yml` vars section. Common overrides include:
- `security_autoupdate_reboot`: Enable/disable automatic reboots for security updates
- `security_autoupdate_reboot_time`: Schedule reboot time (24h format)
- `swap_file_size_mb`: Configure swap file size (from geerlingguy.swap role)

### Security Features Details

**Fail2ban Configuration** (roles/security/templates/jail.local.j2):
- SSH jail: 5 failed attempts = 3600s ban
- SSH DDoS protection: 10 attempts in 60s = 600s ban

**Kernel Hardening** (roles/sysctl/tasks/main.yml):
- SYN flood protection, IP spoofing prevention
- ICMP redirect blocking, martian packet logging
- Kernel pointer and dmesg access restrictions

**Docker Security** (roles/docker/templates/daemon.json.j2):
- Log rotation (10MB max, 3 files)
- Inter-container communication disabled by default
- No new privileges flag enabled
- Userland proxy disabled for performance

**Audit Monitoring** (roles/auditd/tasks/main.yml):
Tracks: sudo usage, SSH config changes, user/group modifications, Docker operations, kernel module loading, time changes