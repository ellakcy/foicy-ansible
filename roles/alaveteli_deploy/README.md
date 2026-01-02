Alaveteli Deploy
================

Performs zero-downtime deployments of Alaveteli using a Capistrano-style release structure.

Description
-----------

This role handles the deployment of new Alaveteli releases. It can be run repeatedly to deploy updates from the git repository without downtime.

The role performs these operations:

1. Clones/updates the git repository to the cached copy
2. Generates a timestamped release directory
3. Copies code from the cached copy to the new release
4. Sets appropriate permissions
5. Creates necessary Rails directories
6. Records the git revision (SHA) in a REVISION file
7. Symlinks shared configuration and data into the release
8. Runs Rails post-deployment tasks (migrations, asset compilation, etc.)
9. Removes the .git directory from the release
10. Updates the `current` symlink to point to the new release
11. Restarts Alaveteli services
12. Cleans up old releases (keeps last 5)

Requirements
------------

- Deployment structure must already be set up (via `alaveteli_deploy_setup` role)
- Git installed on the target system
- Network access to the git repository
- The following directory structure must exist:
  ```
  /var/www/{{ alaveteli_host }}/alaveteli/
  ├── releases/
  ├── shared/
  │   ├── cached-copy/
  │   ├── Configuration files
  │   └── Data directories
  ```

Role Variables
--------------

This role requires the following variables (typically set in inventory or group_vars):

- `alaveteli_host`: The hostname for the Alaveteli installation (e.g., `"foi.com.cy"`)
- `alaveteli_user`: The user that owns the Alaveteli installation (e.g., `"greg"`)
- `alaveteli_group`: The group that owns the Alaveteli installation (e.g., `"greg"`)

Defined in `defaults/main.yml`:

- `repo_url`: Git repository URL (default: `"https://github.com/mysociety/alaveteli.git"`)
- `branch`: Git branch to deploy (default: `"master"`)

Generated variables:

- `timestamp`: Auto-generated timestamp for the release directory (format: `YYYYMMDDHHMMSS`)
- `git_revision`: The Git commit SHA of the deployed release

Dependencies
------------

This role depends on:

- `alaveteli_deploy_setup`: Must have been run once to create the deployment structure

Handlers
--------

This role includes one handler:

- `Restart Alaveteli Services`: Restarts all Alaveteli systemd services (puma, sidekiq, alert-tracks, send-notifications)

Release Management
------------------

### Release Directory Structure

Each deployment creates a new timestamped directory under `releases/`:

```
/var/www/foi.com.cy/alaveteli/
├── releases/
│   ├── 20260102120000/
│   ├── 20260102130000/
│   └── 20260102140000/
├── shared/
└── current -> releases/20260102140000/
```

### Automatic Cleanup

The role automatically keeps only the last 5 releases. Older releases are removed to save disk space.

### Shared Files and Directories

The following are symlinked from the `shared/` directory into each release:

**Configuration files:**
- `config/general.yml`
- `config/database.yml`
- `config/initializers/rails_env.rb`
- `config/storage.yml`
- `config/sidekiq.yml`

**Data directories:**
- `lib/acts_as_xapian/xapiandbs` (search indexes)
- `log` (application logs)
- `tmp/pids` (process IDs)
- `themes` (custom themes)

Example Playbook
----------------

Deploy the latest code from master branch:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
```

Deploy a specific branch:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
      vars:
        branch: "develop"
```

Deploy from a fork:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
      vars:
        repo_url: "https://github.com/your-org/alaveteli-fork.git"
        branch: "production"
```

Deployment Workflow
-------------------

### Initial Setup (One-time)

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
    - role: alaveteli_deploy  # First deployment
```

### Subsequent Deployments

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
```

Or create a separate deployment playbook:

```yaml
# deploy.yml
---
- name: Deploy Alaveteli
  hosts: foi.com.cy
  become: yes
  roles:
    - alaveteli_deploy
```

Then run:
```bash
ansible-playbook -i inventory.yml deploy.yml
```

Post-Deploy Script
------------------

The role runs `/script/rails-post-deploy` which typically performs:

- Installing/updating Ruby gem dependencies
- Running database migrations
- Precompiling assets
- Clearing caches
- Any other deployment tasks defined in the Alaveteli codebase

Zero-Downtime Deployment
------------------------

The deployment achieves zero-downtime by:

1. Preparing the new release completely before switching to it
2. Atomically updating the `current` symlink to point to the new release
3. Only then restarting the application services

The services reload with the new code from the `current` symlink without taking the site offline.

Rollback
--------

To rollback to a previous release, manually update the `current` symlink:

```bash
# SSH to the server
cd /var/www/foi.com.cy/alaveteli
ls -lt releases/  # Find the previous release

# Update symlink to previous release
ln -sfn /var/www/foi.com.cy/alaveteli/releases/20260102120000 current

# Restart services
sudo systemctl restart alaveteli.puma.service
sudo systemctl restart alaveteli.sidekiq.service
```

Troubleshooting
---------------

### Check Deployment Status

```bash
# View current release
ssh foi.com.cy "ls -la /var/www/foi.com.cy/alaveteli/current"

# View recent releases
ssh foi.com.cy "ls -lt /var/www/foi.com.cy/alaveteli/releases"

# Check git revision
ssh foi.com.cy "cat /var/www/foi.com.cy/alaveteli/current/REVISION"
```

### View Deployment Logs

```bash
# Application logs
ssh foi.com.cy "tail -f /var/www/foi.com.cy/alaveteli/shared/log/production.log"

# Service status
ssh foi.com.cy "sudo systemctl status alaveteli.puma.service"
```

### Failed Deployment

If a deployment fails:

1. The `current` symlink still points to the previous working release
2. The failed release directory remains in `releases/` for investigation
3. Services are still running the previous release
4. Fix the issue and run the deployment again

Notes
-----

- Deployments typically take 2-5 minutes depending on code changes
- The git repository is cached locally for faster deployments
- Database migrations run automatically during deployment
- Asset compilation can be time-consuming for large changes
- Services are restarted automatically after deployment

License
-------

MIT-0

Author Information
------------------

Created for foi.com.cy Alaveteli deployment automation.
