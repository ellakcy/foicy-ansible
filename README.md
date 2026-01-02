# Foicy Ansible

Ansible automation for deploying and managing Alaveteli Freedom of Information request platform on foi.com.cy.

## Overview

This repository contains Ansible playbooks and roles for automated deployment of [Alaveteli](https://github.com/mysociety/alaveteli), an open-source platform for Freedom of Information requests developed by mySociety.

The automation implements a production-ready deployment using a Capistrano-style structure with timestamped releases, shared directories, and zero-downtime deployments.

## Quick Start

### Prerequisites

- Ansible 2.9 or higher installed on your control machine
- SSH access to the target Ubuntu server
- Sudo/root privileges on the target server

### Initial Setup (One-time)

Run the complete installation and setup:

```bash
ansible-playbook -i inventory.yml foicy_initial_install.yml
```

This performs:
1. System updates
2. Alaveteli installation
3. Deployment structure setup
4. First deployment

### Subsequent Deployments

After initial setup, deploy updates using:

```bash
ansible-playbook -i inventory.yml deploy.yml
```

## Repository Structure

```
foicy-ansible/
├── inventory.yml                           # Host inventory
├── foicy_initial_install.yml               # Initial setup playbook (run once)
├── deploy.yml                              # Deployment playbook (run repeatedly)
├── roles/
│   ├── ubuntu_update/                      # System updates role
│   ├── alaveteli_install/                  # Initial Alaveteli installation
│   ├── alaveteli_deploy_setup/             # One-time deployment structure setup
│   └── alaveteli_deploy/                   # Repeatable deployment role
└── CLAUDE.md                               # Development guide for Claude Code
```

## Roles

### 1. ubuntu_update

Updates and upgrades all Ubuntu packages and reboots if required.

- Updates apt package cache
- Performs distribution upgrade
- Conditionally reboots the system

**Documentation**: [roles/ubuntu_update/README.md](roles/ubuntu_update/README.md)

### 2. alaveteli_install

Installs Alaveteli using the official mySociety installation script.

- Downloads and executes the mySociety install script
- Sets up all dependencies (Ruby, PostgreSQL, Xapian, nginx, postfix, etc.)
- Performs initial system configuration
- Reboots to finalize installation

**Run once during initial setup.**

**Documentation**: [roles/alaveteli_install/README.md](roles/alaveteli_install/README.md)

### 3. alaveteli_deploy_setup

One-time setup that converts the basic installation to a Capistrano-style deployment structure.

- Creates releases/shared/current directory structure
- Migrates configuration and data to shared directories
- Creates cached-copy directory for git repository
- Updates system service configurations to use `/current/` path

**Run once during initial setup.**

**Documentation**: [roles/alaveteli_deploy_setup/README.md](roles/alaveteli_deploy_setup/README.md)

### 4. alaveteli_deploy

Performs zero-downtime deployments of Alaveteli releases.

- Clones/updates git repository
- Creates timestamped release
- Links shared files/directories
- Runs Rails post-deployment tasks
- Updates `current` symlink
- Restarts services
- Cleans up old releases (keeps last 5)

**Run repeatedly for each deployment.**

**Documentation**: [roles/alaveteli_deploy/README.md](roles/alaveteli_deploy/README.md)

## Playbooks

### foicy_initial_install.yml

Complete initial setup workflow. **Run once.**

```yaml
- name: Alavateli initial install for foi.com.cy
  hosts: foi.com.cy
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
    - role: alaveteli_deploy
```

### deploy.yml

Deployment workflow for updates. **Run for each deployment.**

```yaml
- name: Deploy Alaveteli
  hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
```

## Inventory Configuration

The inventory is defined in [inventory.yml](inventory.yml):

```yaml
all:
  hosts:
    foi.com.cy:
      ansible_host: 116.203.48.195
      ansible_python_interpreter: /usr/bin/python3
      alaveteli_host: foi.com.cy
      alaveteli_user: greg
      alaveteli_group: greg
```

### Key Variables

- `ansible_host`: IP address or hostname of the target server
- `ansible_python_interpreter`: Python interpreter path on the target
- `alaveteli_host`: Hostname for the Alaveteli installation
- `alaveteli_user`: System user that owns the installation
- `alaveteli_group`: System group that owns the installation

## Deployment Structure

After running initial setup, the deployment structure will be:

```
/var/www/foi.com.cy/alaveteli/
├── releases/
│   ├── 20260102120000/              # Timestamped releases
│   ├── 20260102130000/
│   └── 20260102140000/
├── shared/
│   ├── general.yml                  # Configuration files
│   ├── database.yml
│   ├── rails_env.rb
│   ├── storage.yml
│   ├── sidekiq.yml
│   ├── cache/                       # Persistent data
│   ├── files/
│   ├── xapiandbs/
│   ├── log/
│   ├── tmp/pids/
│   ├── themes/
│   └── cached-copy/                 # Git repository cache
├── current -> releases/20260102140000/  # Symlink to active release
└── alaveteli.bak/                   # Backup of original installation
```

## Common Operations

### Deploy Latest Code

```bash
ansible-playbook -i inventory.yml deploy.yml
```

### Deploy Specific Branch

```bash
ansible-playbook -i inventory.yml deploy.yml -e "branch=develop"
```

### Deploy from Fork

```bash
ansible-playbook -i inventory.yml deploy.yml \
  -e "repo_url=https://github.com/your-org/alaveteli-fork.git" \
  -e "branch=production"
```

### Check Mode (Dry Run)

Preview changes without applying them:

```bash
ansible-playbook -i inventory.yml deploy.yml --check
```

### Limit to Specific Hosts

Run only on specific hosts:

```bash
ansible-playbook -i inventory.yml deploy.yml --limit foi.com.cy
```

## System Services

The deployment configures and manages these system services:

- **nginx**: Web server (config: `/etc/nginx/sites-available/foi.com.cy`)
- **postfix**: Mail handling (config: `/etc/postfix/master.cf`)
- **cron**: Scheduled tasks (config: `/etc/cron.d/alaveteli`)
- **systemd services**:
  - `alaveteli.puma.service` - Web application server
  - `alaveteli.sidekiq.service` - Background job processor
  - `alaveteli.alert-tracks.service` - Email alert tracking
  - `alaveteli.send-notifications.service` - Notification sender

## Deployment Workflow

### First Time Setup

```bash
# 1. Configure inventory.yml with your server details
# 2. Run initial setup (this will take 15-30 minutes)
ansible-playbook -i inventory.yml foicy_initial_install.yml
```

### Regular Deployments

```bash
# Deploy latest changes from master branch
ansible-playbook -i inventory.yml deploy.yml
```

### Release Management

- Each deployment creates a timestamped release in `releases/`
- The `current` symlink always points to the active release
- Old releases are automatically cleaned up (keeps last 5)
- Services are restarted to load the new code

### Rollback

To rollback to a previous release:

```bash
# SSH to server
ssh foi.com.cy

# View available releases
ls -lt /var/www/foi.com.cy/alaveteli/releases/

# Update symlink to previous release
sudo ln -sfn /var/www/foi.com.cy/alaveteli/releases/20260102120000 \
  /var/www/foi.com.cy/alaveteli/current

# Restart services
sudo systemctl restart alaveteli.puma.service
sudo systemctl restart alaveteli.sidekiq.service
```

## Troubleshooting

### Check Service Status

```bash
ssh foi.com.cy "sudo systemctl status alaveteli.puma.service"
ssh foi.com.cy "sudo systemctl status alaveteli.sidekiq.service"
```

### View Logs

```bash
# Application logs
ssh foi.com.cy "sudo tail -f /var/www/foi.com.cy/alaveteli/shared/log/production.log"

# Deployment logs
ssh foi.com.cy "sudo journalctl -u alaveteli.puma.service -f"
```

### Verify Deployment

```bash
# Check current release
ssh foi.com.cy "ls -la /var/www/foi.com.cy/alaveteli/current"

# Check git revision
ssh foi.com.cy "cat /var/www/foi.com.cy/alaveteli/current/REVISION"

# View recent releases
ssh foi.com.cy "ls -lt /var/www/foi.com.cy/alaveteli/releases/"
```

### Failed Deployment

If a deployment fails:

1. The `current` symlink still points to the previous working release
2. Services continue running the previous release
3. The failed release directory remains for investigation
4. Fix the issue and run deployment again

## Customization

### Different Git Repository

Edit [roles/alaveteli_deploy/defaults/main.yml](roles/alaveteli_deploy/defaults/main.yml):

```yaml
repo_url: "https://github.com/your-org/alaveteli-fork.git"
branch: "production"
```

### Keep More/Fewer Releases

Edit the cleanup task in [roles/alaveteli_deploy/tasks/main.yml](roles/alaveteli_deploy/tasks/main.yml):

```yaml
# Change tail -n +6 to keep different number (current keeps 5)
ls -1t | tail -n +11 | xargs -r rm -rf  # Keeps 10 releases
```

## License

MIT-0

## Contributing

This is a specific deployment automation for foi.com.cy. For general Alaveteli documentation and contributions, see the [official Alaveteli repository](https://github.com/mysociety/alaveteli).

## Additional Resources

- [Alaveteli Official Documentation](https://alaveteli.org/)
- [mySociety](https://www.mysociety.org/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Capistrano Deployment Pattern](https://capistranorb.com/)

## Author

Created for foi.com.cy Alaveteli deployment automation.
