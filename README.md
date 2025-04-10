# GitHub Enterprise Server Backup Utilities Ansible Playbook

This Ansible playbook automates the installation, configuration, and scheduling of GitHub Enterprise Server Backup Utilities on a dedicated backup host. It follows best practices for securely managing GitHub Enterprise Server backups with version compatibility support and secure credential management via Azure Key Vault.

## Overview

GitHub Enterprise Server Backup Utilities is a backup system you install on a separate host from your GitHub Enterprise Server instance. It takes backup snapshots of your GitHub Enterprise Server instance at regular intervals over a secure SSH network connection.

This playbook automates the entire lifecycle of backup utilities:
- Installing the correct version of backup utilities compatible with your GHE instance
- Configuring the backup utilities with secure best practices
- Setting up automated scheduled backups with proper rotation

## Project Structure

```
.
├── ansible.cfg                    # Ansible configuration file
├── inventory/                     # Inventory directory
│   ├── group_vars/                # Group variables
│   │   ├── all.yml                # Common variables for all hosts
│   │   └── backup_servers.yml     # Variables specific to backup servers
│   ├── host_vars/                 # Host-specific variables
│   │   └── backup-host.yml        # Variables for specific backup hosts
│   └── hosts                      # Inventory file
├── playbooks/                     # Playbooks directory
│   └── github/                    # GitHub-related playbooks
│       ├── main.yml               # Main playbook for full setup
│       └── update.yml             # Playbook for updates only
├── requirements.yml               # Required collections and roles
└── roles/                         # Roles directory
    ├── azure_credentials/         # Role for Azure Key Vault integration
    ├── backup_utils_install/      # Role for installing backup utilities
    ├── backup_utils_configure/    # Role for configuring backup utilities
    └── backup_utils_schedule/     # Role for scheduling automated backups
```

## Prerequisites

- A dedicated backup host running Linux (Ubuntu is recommended)
- Network connectivity between the backup host and GitHub Enterprise Server
- Azure Key Vault for secure credential storage (optional, but recommended)
- Ansible 2.10+ on your control node

## Required Packages

The following packages will be installed on the backup host:
- bash
- git
- rsync
- moreutils
- gawk
- jq
- bc

## Roles

### azure_credentials

This role retrieves SSH credentials from Azure Key Vault for connecting to your GitHub Enterprise Server instance:

- Uses Azure Managed Identity if enabled on the VM
- Falls back to service principal authentication if MSI is not available
- Stores the SSH key securely on the backup host

### backup_utils_install

This role handles the installation of GitHub Enterprise Server Backup Utilities:

- Checks compatibility between backup utilities and your GHE instance
- Downloads and installs the correct version of backup utilities
- Preserves existing configuration when upgrading
- Creates necessary users and directories with proper permissions

### backup_utils_configure

This role configures the backup utilities:

- Creates a proper backup.config file with your settings
- Sets up SSH connectivity to your GitHub Enterprise Server
- Creates necessary directory structure for backups
- Validates connectivity to ensure everything is working

### backup_utils_schedule

This role sets up automated scheduled backups:

- Supports both systemd timers and cron jobs for scheduling
- Creates a backup wrapper script with logging and notification support
- Implements grandfather-father-son backup rotation scheme
- Configures log rotation and error handling

## Azure Key Vault Integration

This playbook integrates with Azure Key Vault to securely retrieve GitHub Enterprise Server SSH credentials:

1. It uses Azure Managed Identity when available (recommended)
2. Falls back to service principal authentication if MSI is not available
3. Supports retrieving both the SSH username and private key
4. Never stores credentials in the playbook or variables files

## Usage

### Initial Setup

To perform a complete setup including installation, configuration, and scheduling:

```bash
ansible-playbook playbooks/github/main.yml
```

### Updating Backup Utilities

To update an existing installation to a newer version:

```bash
ansible-playbook playbooks/github/update.yml
```

Or using tags:

```bash
ansible-playbook playbooks/github/main.yml --tags install -e "upgrade_backup_utils=true"
```

### Reconfiguring Without Reinstalling

To update configuration without reinstalling:

```bash
ansible-playbook playbooks/github/main.yml --tags configure
```

### Running Individual Roles

You can run individual roles using tags:

```bash
ansible-playbook playbooks/github/main.yml --tags "install"
ansible-playbook playbooks/github/main.yml --tags "configure"
ansible-playbook playbooks/github/main.yml --tags "schedule"
```

## Configuration

The playbook is highly configurable through variables. Key settings include:

### Backup Host Settings

- `backup_utils_user`: User to run backup utilities (default: backup)
- `backup_utils_install_dir`: Installation directory (default: /opt/github-backup-utils)
- `backup_utils_data_dir`: Directory for storing backups (default: /data/github-backups)

### GitHub Enterprise Server Settings

- `ghe_hostname`: Hostname of your GitHub Enterprise Server
- `ghe_port`: SSH port (default: 122)
- `ghe_ssh_user`: SSH username for connecting to GHE

### Backup Schedule Settings

- `backup_schedule_enabled`: Enable scheduled backups (default: true)
- `backup_schedule_hour`: Hour to run backups (default: */6 - every 6 hours)
- `backup_schedule_minute`: Minute to run backups (default: 0)

### Azure Key Vault Settings

- `azure_keyvault_url`: URL of your Azure Key Vault
- `azure_managed_identity_enabled`: Whether to use Azure Managed Identity
- `ghe_ssh_user_secret_name`: Name of the secret containing SSH username
- `ghe_ssh_key_secret_name`: Name of the secret containing SSH private key

## Version Compatibility

This playbook ensures compatibility between your GitHub Enterprise Server instance and the backup utilities:

- Backup utilities must be at least the same version as your GHE instance
- Backup utilities cannot be more than two versions ahead of your GHE instance
- The playbook automatically detects your GHE version and installs the correct backup utilities version

## Security Considerations

- All credentials are retrieved from Azure Key Vault at runtime
- SSH keys are stored with restrictive permissions (0600)
- Backup data directories are created with proper permissions
- Backup scripts run with least privilege

## Backup Rotation

The playbook implements a grandfather-father-son backup rotation scheme:

- Daily backups: Kept for 7 days by default
- Weekly backups: Kept for 4 weeks by default
- Monthly backups: Kept for 6 months by default

## Maintenance

- Updates to newer versions can be done with a dedicated update playbook
- Configuration changes can be made by editing the group_vars or host_vars files
- All operations are idempotent and can be safely re-run

## Troubleshooting

Check the following logs for troubleshooting:

- Backup logs: `/var/log/github-backup/backup-*.log`
- Rotation logs: `/var/log/github-backup/rotation-*.log`
- Ansible logs: Generated during playbook execution

## License

This Ansible playbook is provided under the MIT license.

## References

- [GitHub Enterprise Server Backup Utilities Documentation](https://github.com/github/backup-utils/blob/master/README.md)
- [GitHub Enterprise Server Documentation](https://docs.github.com/en/enterprise-server)
- [Ansible Documentation](https://docs.ansible.com/)
- [Azure Key Vault Documentation](https://docs.microsoft.com/en-us/azure/key-vault/)
