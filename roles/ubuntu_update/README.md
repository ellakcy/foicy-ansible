Ubuntu Update
=============

Updates and upgrades all packages on Ubuntu systems and reboots if required.

Description
-----------

This role performs a complete system update and upgrade on Ubuntu-based systems. It's designed to be run as a preliminary step before installing or deploying applications to ensure the system has the latest security patches and package updates.

The role performs these operations:

1. Updates the apt package cache
2. Performs a distribution upgrade of all packages
3. Removes obsolete packages (autoremove)
4. Cleans the package cache (autoclean)
5. Checks if a reboot is required
6. Reboots the system if needed (conditionally)

Requirements
------------

- Ubuntu-based system (tested on Ubuntu 20.04, 22.04, and 24.04)
- Sudo/root access on the target system
- Internet connection to download package updates

Role Variables
--------------

This role has no configurable variables. All operations are performed using sensible defaults.

Dependencies
------------

None. This is typically the first role in a deployment workflow.

Example Playbook
----------------

Basic usage:

```yaml
- hosts: all
  become: yes
  roles:
    - role: ubuntu_update
```

As part of a deployment workflow:

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: ubuntu_update
    - role: alaveteli_install
    - role: alaveteli_deploy_setup
```

Limited to specific hosts:

```yaml
- hosts: production
  become: yes
  roles:
    - role: ubuntu_update
```

Behavior
--------

### Package Updates

The role uses `apt upgrade: dist` which performs a distribution upgrade, intelligently handling package dependencies and removing obsolete packages when necessary.

### Conditional Reboot

The role checks for the existence of `/var/run/reboot-required` to determine if a reboot is needed. This file is created by Ubuntu when package updates require a system restart (typically for kernel updates or core system libraries).

If a reboot is required:
- The system will reboot automatically
- Ansible will wait for the system to come back online
- The playbook will continue after the reboot completes

If no reboot is required:
- The reboot task is skipped
- The playbook continues immediately

Notes
-----

- This role is idempotent and safe to run multiple times
- Package updates may take several minutes depending on the number of packages
- A reboot will interrupt any active connections to the system
- After a reboot, it may take 30-60 seconds for the system to be fully ready
- Consider running this role during maintenance windows for production systems

License
-------

MIT-0

Author Information
------------------

Created for foi.com.cy Alaveteli deployment automation.
