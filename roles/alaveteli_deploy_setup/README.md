Alaveteli Deploy Setup
======================

One-time setup that converts a standard Alaveteli installation into a Capistrano-style deployment structure.

Description
-----------

This role performs the one-time setup required to enable repeatable, zero-downtime deployments of Alaveteli. It transforms a basic Alaveteli installation (created by the `alaveteli_install` role) into a production-ready deployment structure.

**This role should only be run once** during initial setup. For subsequent deployments, use the `alaveteli_deploy` role.

The role performs these key operations:

1. Installs the ACL package for proper permission handling
2. Backs up the original installation to `alaveteli.bak`
3. Creates a Capistrano-style directory structure with `releases/`, `shared/`, and `current/` symlink
4. Migrates configuration files and data from the original installation to the shared directory
5. Creates the cached-copy directory for git repository caching
6. Updates system service configurations (nginx, postfix, cron, systemd) to use the `/current/` path

After running this role, use the `alaveteli_deploy` role to perform actual deployments.

Requirements
------------

- Ubuntu-based system with Alaveteli already installed (via `alaveteli_install` role)
- Nginx, Postfix, and systemd services configured for Alaveteli
- Sufficient disk space for releases directory

Role Variables
--------------

This role requires the following variables (typically set in inventory or group_vars):

- `alaveteli_host`: The hostname for the Alaveteli installation (e.g., `"foi.com.cy"`)
- `alaveteli_user`: The user that owns the Alaveteli installation (e.g., `"greg"`)
- `alaveteli_group`: The group that owns the Alaveteli installation (e.g., `"greg"`)

This role has no default variables.

Directory Structure Created
----------------------------

```
/var/www/{{ alaveteli_host }}/alaveteli/
├── releases/                     # Empty initially; deployments create timestamped releases here
├── shared/
│   ├── general.yml               # Alaveteli configuration (migrated from original install)
│   ├── database.yml              # Database configuration
│   ├── rails_env.rb              # Rails environment settings
│   ├── storage.yml               # ActiveStorage configuration
│   ├── sidekiq.yml               # Background job configuration
│   ├── cache/                    # Rails cache directory
│   ├── files/                    # User-uploaded files
│   ├── xapiandbs/                # Xapian search indexes
│   ├── log/                      # Application logs
│   ├── tmp/pids/                 # Process ID files
│   ├── themes/                   # Custom themes
│   └── cached-copy/              # Git repository cache (empty until first deploy)
└── alaveteli.bak/                # Backup of original installation
```

The `current/` symlink is created during the first deployment (via `alaveteli_deploy` role).

Dependencies
------------

This role depends on:

- `alaveteli_install`: Must be run first to create the initial Alaveteli installation

Handlers
--------

This role includes three handlers that reload services after configuration changes:

- `Reload Nginx`: Reloads nginx configuration
- `Reload Postfix`: Reloads postfix configuration
- `Reload Systemd`: Reloads systemd daemon configuration

Example Playbook
----------------

Initial setup workflow:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
    - role: alaveteli_deploy  # First deployment
```

**Note**: This role should only appear in your initial setup playbook, not in your regular deployment playbooks.

System Services Updated
-----------------------

This role updates configuration files for the following services to use the `/current/` symlink path:

- **Nginx**: `/etc/nginx/sites-available/{{ alaveteli_host }}`
- **Cron**: `/etc/cron.d/alaveteli`
- **Postfix**: `/etc/postfix/master.cf`
- **Systemd services**:
  - `alaveteli.alert-tracks.service`
  - `alaveteli.puma.service`
  - `alaveteli.send-notifications.service`
  - `alaveteli.sidekiq.service`

After this setup, these services will always use the code from the `current` symlink, which points to the active release.

Migration Details
-----------------

The role migrates the following from the original installation to the shared directory:

**Configuration files:**
- `config/general.yml` → `shared/general.yml`
- `config/database.yml` → `shared/database.yml`
- `config/rails_env.rb` → `shared/rails_env.rb`
- `config/storage.yml` → `shared/storage.yml`
- `config/sidekiq.yml` → `shared/sidekiq.yml`

**Data directories:**
- `cache/` → `shared/cache/`
- `files/` → `shared/files/`
- `lib/acts_as_xapian/xapiandbs/` → `shared/xapiandbs/`
- `log/` → `shared/log/`

These files and directories persist across all deployments and are symlinked into each release.

Next Steps
----------

After running this role:

1. Verify the directory structure was created correctly
2. Run the `alaveteli_deploy` role to perform the first deployment
3. For all future deployments, only run the `alaveteli_deploy` role

Notes
-----

- **This is a one-time operation** - do not run this role multiple times
- The original installation is preserved in `alaveteli.bak` as a backup
- The role is idempotent - it checks for existing files/directories before creating or moving them
- System services are reconfigured but not restarted (they continue running during setup)
- After setup completes, you must run `alaveteli_deploy` to create the first release

License
-------

MIT-0

Author Information
------------------

Created for foi.com.cy Alaveteli deployment automation.
