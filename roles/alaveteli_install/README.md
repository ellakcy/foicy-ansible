Alaveteli Install
=================

Installs Alaveteli Freedom of Information request platform using the mySociety installation script.

Description
-----------

This role automates the initial installation of Alaveteli by downloading and executing the official mySociety installation script. It handles the complete setup process and performs a system reboot to finalize the installation.

The role performs these steps:

1. Downloads the Alaveteli installation script from mySociety's repository
2. Executes the installation script with specified parameters
3. Cleans up the temporary installation script
4. Reboots the system to complete the installation

After this role completes, you'll have a basic Alaveteli installation at `/var/www/{{ alaveteli_host }}/alaveteli/`. For production deployments, follow this role with `alaveteli_deploy_setup` to convert the installation to a Capistrano-style deployment structure.

Requirements
------------

- Ubuntu-based system (tested on Ubuntu 20.04 and 22.04)
- Sudo/root access on the target system
- Internet connection to download the installation script
- Sufficient disk space for Alaveteli and its dependencies

Role Variables
--------------

This role requires the following variables to be set (typically in inventory or group_vars):

- `alaveteli_host`: The hostname for the Alaveteli installation (e.g., `"foi.com.cy"`)
- `alaveteli_user`: The user that will own the Alaveteli installation (e.g., `"greg"`)
- `alaveteli_group`: The group that will own the Alaveteli installation (e.g., `"greg"`)

Example inventory configuration:

```yaml
all:
  hosts:
    foi.com.cy:
      ansible_host: 116.203.48.195
      alaveteli_host: foi.com.cy
      alaveteli_user: greg
      alaveteli_group: greg
```

Dependencies
------------

None. This is typically the first role in an Alaveteli deployment workflow.

Installation Process
--------------------

The role uses the official mySociety installation script which:

- Installs all required system packages (Ruby, PostgreSQL, Xapian, etc.)
- Sets up the database
- Configures web server (nginx)
- Configures mail handling (postfix)
- Creates systemd service files
- Performs initial Alaveteli setup

Example Playbook
----------------

Basic usage:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: alaveteli_install
```

Complete workflow with deployment setup:

```yaml
- hosts: foi.com.cy
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
```

Notes
-----

- This role will trigger a system reboot
- The installation process may take several minutes depending on system resources
- After installation, Alaveteli will be accessible at the configured hostname
- For production use, follow this role with `alaveteli_deploy_setup` for proper deployment structure
- The role is idempotent - it will skip installation if Alaveteli is already installed

License
-------

MIT-0

Author Information
------------------

Created for foi.com.cy Alaveteli deployment automation.
