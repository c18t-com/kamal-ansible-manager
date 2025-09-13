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
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini --ask-pass playbook.yml
```

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
2. `packages` - Updates system packages and installs essential tools
3. `docker` - Installs and configures Docker
4. `deployer` - Creates deployer user with SSH keys for Kamal deployments
5. `firewall` - Configures UFW firewall rules
6. `security` - Sets up fail2ban, disables SSH password auth, configures unattended upgrades
7. `geerlingguy.swap` - External role for swap file configuration
8. `reboot_if_needed` - Reboots server if kernel updates require it

### Configuration Files
- `hosts.ini` - Ansible inventory defining target servers in `webservers` group
- `requirements.yml` - External Ansible collections and roles dependencies
- Variables are configured directly in `playbook.yml` under the `vars` section

### Key Features
- **Idempotent operations**: Roles check for existing configurations before making changes
- **Security hardening**: Removes snap, configures fail2ban, UFW, and disables SSH password login
- **Kamal preparation**: Creates dedicated deployer user with SSH key management
- **Optional cloud provisioning**: Scaleway integration for automated server creation

### SSH Key Management
The deployer role generates ed25519 SSH keys and stores private keys locally in `./keys/{{ inventory_hostname }}/deployer_id_ed25519` for use with Kamal deployments.

### Variable Configuration
Override default settings in `playbook.yml` vars section. Common overrides include:
- `security_autoupdate_reboot`: Enable/disable automatic reboots for security updates
- `security_autoupdate_reboot_time`: Schedule reboot time (24h format)
- `swap_file_size_mb`: Configure swap file size (from geerlingguy.swap role)