Alaveteli Config
================

Manages Alaveteli configuration settings in the shared `general.yml` file.

Description
-----------

This role updates the Alaveteli configuration file (`general.yml`) with custom settings for your deployment. It modifies the shared configuration file that persists across all releases.

The role currently supports configuring:

- **SITE_NAME**: The name of your FOI site (e.g., "FOI Cyprus")

Additional configuration options can be easily added as needed.

Requirements
------------

- Alaveteli must be installed and the deployment structure must be set up
- The `general.yml` file must exist in the shared directory (`/var/www/{{ alaveteli_host }}/alaveteli/shared/general.yml`)

This role should be run:
- After `alaveteli_deploy_setup` (which creates the shared directory structure)
- Before or after `alaveteli_deploy` (configuration changes will be picked up by the next deployment)

Role Variables
--------------

This role requires the following variables (typically set in inventory or group_vars):

- `alaveteli_host`: The hostname for the Alaveteli installation (e.g., `"foi.com.cy"`)

Defined in `defaults/main.yml`:

- `alaveteli_site_name`: The site name to display (default: `"FOI Cyprus"`)

You can override these in your inventory or playbook:

```yaml
alaveteli_site_name: "My Custom FOI Site"
```

Dependencies
------------

None. This role can be run independently once the deployment structure exists.

Handlers
--------

This role includes one handler:

- `Restart Alaveteli Services`: Restarts all Alaveteli systemd services to apply configuration changes

Configuration File Location
----------------------------

The configuration file is located at:
```
/var/www/{{ alaveteli_host }}/alaveteli/shared/general.yml
```

This file is symlinked into each release at `config/general.yml`, so changes persist across deployments.

Example Playbook
----------------

Basic usage (updates site name to "FOI Cyprus"):

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_config
```

Custom site name:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_config
      vars:
        alaveteli_site_name: "Freedom of Information Portal"
```

As part of deployment workflow:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_deploy
    - role: alaveteli_config
```

Or in initial setup:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
    - role: alaveteli_config    # Configure before first deployment
    - role: alaveteli_deploy
```

Safety Features
---------------

- **Existence check**: The role checks if `general.yml` exists before attempting modifications
- **Backup**: Creates a backup of `general.yml` before making changes (`.bak` file)
- **Idempotent**: Safe to run multiple times - only updates if the value has changed
- **Service restart**: Automatically restarts services to apply configuration changes

Adding More Configuration Options
----------------------------------

To add more configuration settings, update the tasks file:

```yaml
# In roles/alaveteli_config/tasks/main.yml
- name: Update DOMAIN in general.yml
  ansible.builtin.lineinfile:
    path: "/var/www/{{ alaveteli_host }}/alaveteli/shared/general.yml"
    regexp: "^DOMAIN:"
    line: "DOMAIN: '{{ alaveteli_domain }}'"
    backup: yes
  when: general_yml_stat.stat.exists
  become: true
  notify: Restart Alaveteli Services
```

And add the variable to defaults:

```yaml
# In roles/alaveteli_config/defaults/main.yml
alaveteli_domain: "foi.com.cy"
```

Reference
---------

For all available configuration options, see the Alaveteli example configuration:
https://github.com/mysociety/alaveteli/blob/develop/config/general.yml-example

Common settings you might want to configure:

- `SITE_NAME`: Site display name
- `DOMAIN`: Primary domain name
- `CONTACT_EMAIL`: Contact email address
- `LOCALE`: Default language/locale
- `TIME_ZONE`: Server timezone
- `INCOMING_EMAIL_DOMAIN`: Domain for receiving FOI request emails
- `INCOMING_EMAIL_PREFIX`: Email prefix for incoming requests

Troubleshooting
---------------

### Configuration not applied

If configuration changes don't appear:

1. Check that the file was updated:
   ```bash
   ssh foi.com.cy "grep SITE_NAME /var/www/foi.com.cy/alaveteli/shared/general.yml"
   ```

2. Verify services were restarted:
   ```bash
   ssh foi.com.cy "sudo systemctl status alaveteli.puma.service"
   ```

3. Check the symlink in the current release:
   ```bash
   ssh foi.com.cy "ls -la /var/www/foi.com.cy/alaveteli/current/config/general.yml"
   ```

### File not found

If you get a "file not found" warning:

1. Ensure `alaveteli_deploy_setup` has been run
2. Verify the shared directory exists:
   ```bash
   ssh foi.com.cy "ls -la /var/www/foi.com.cy/alaveteli/shared/"
   ```

### Backup files

Backups are created with a `.bak` extension. To view or restore:

```bash
# View backup
ssh foi.com.cy "cat /var/www/foi.com.cy/alaveteli/shared/general.yml.bak"

# Restore from backup
ssh foi.com.cy "sudo cp /var/www/foi.com.cy/alaveteli/shared/general.yml.bak \
  /var/www/foi.com.cy/alaveteli/shared/general.yml"
```

Notes
-----

- Configuration changes require a service restart to take effect
- The role creates a backup before each modification
- Changes persist across deployments (stored in shared directory)
- Always test configuration changes in a staging environment first
- Sensitive values (passwords, API keys) should be stored in Ansible Vault

License
-------

MIT-0

Author Information
------------------

Created for foi.com.cy Alaveteli deployment automation.
