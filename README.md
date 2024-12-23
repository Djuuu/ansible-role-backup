Ansible Role: Backup
====================

Simple(-ish) backup script generator.

Based on https://github.com/geerlingguy/ansible-role-backup

Requirements
------------

Requires the following to be installed:
- rsync
- cron

PostgreSQL database (native or dockerized) needs to be installed if you'd like to enable PostgreSQL database backups.

It's also assumed you have a server running somewhere that can accept backup data via Rsync, and on this backup server, you need to install rsync, and configure accounts with SSH authentication that allows this role to deliver backups to a specific directory via SSH.

Role Variables
--------------

Available role variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# User under which backup jobs will run.
backup_user: "{{ ansible_env.SUDO_USER | default(ansible_env.USER, true) | default(ansible_user_id, true) }}"
backup_group: "{{ ansible_env.SUDO_GROUP | default(ansible_env.GROUP, true) | default(ansible_user_gid, true) }}"

# Should backup use sudo?
backup_privileged: false

# Path to where backups configuration will be stored.
backup_conf_path: /home/{{ backup_user }}/backup
backup_conf_name: backup

# Backup mode
# - rsync: uploads to remote server
# - archive: archive locally
backup_mode: rsync

# rsync options
backup_rsync_path:    /usr/bin/rsync
backup_rsync_options:
  - --archive  # -a
  - --stats
  - --human-readable # -h
backup_rsync_additional_options: []
backup_rsync_compress: true
backup_rsync_delete: true

# Directory for backup logs
backup_log_base_path: "{{ backup_conf_path }}/log"

# Directories to back up. {{ backup_user }} must have read access to these dirs.
backup_directories: []

# Items to exclude from backups. (Use rsync 'excludes' file syntax.)
backup_exclude_items:
  - '.DS_Store'
  - 'Thumbs.db'
  - 'lost+found'

backup_exclude_name: .backup                    # common exclude file: .backup-{{ backup_mode }}-exclude.txt
#backup_exclude_name: ".{{ backup_conf_name }}" # specific exclude file

# Options to control where the backup is delivered.
backup_remote_connection: user@backup.example.com
backup_remote_connection_ssh_options: ''
backup_remote_base_path: "~/backups"
backup_identifier: # optional subdirectory

# Add your server's host key to ensure seamless backup functionality.
backup_remote_host_name: ''
backup_remote_host_key: ''

# Cron options
backup_cron_minute: '0'
backup_cron_hour:   '3'
backup_cron_day:    '*'
backup_cron_month:  '*'
backup_cron_dow:    '7'
backup_cron_use_file: true
backup_cron_state: absent
# Cron state for wrapper script (see wrapper-script.yml task)
backup_cron_wrapper_state: absent
```

A PostgreSQL database backup script can be generated using the `postgres.yml` task and `backup_postgres_` variables.
```yaml
# PostgreSQL backup options
backup_postgres_databases: []
backup_postgres_base_path: "{{ backup_conf_path }}/db"
backup_postgres_options:
  - --no-password # -w
  - --format=p    # -F c|d|t|p (custom, directory, tar, plain text (default))

# Dockerized PostgreSQL options
backup_postgres_dockerized:       false
backup_postgres_docker_container: postgres
```

### Multiple backup configurations

Multiple backup configurations can be defined, using the `multi.yml` task and the `backup_items` variable.

Any variable can be used in a backup item (except for the `backup_postgres_` ones).

```yaml
backup_items:
    
  - backup_directories: [ '/some/directory', '/other/directory' ]
    backup_remote_base_path: /var/backup/main

  - backup_directories: [ '/another/directory' ]
    backup_remote_base_path: /var/backup/other
```

Dependencies
------------

None.

Example Playbook
----------------

### Simple backup with single configuration:

```yaml
- hosts: myhost
  gather_facts: true
  gather_subset:
    - "!all"
    - "!min"
    - user

  roles:
    - djuuu.backup
```

This will create a single `backup.sh` script.


### Multiple backup scripts configuration:

```yaml
- hosts: myhost
  gather_facts: true
  gather_subset:
    - "!all"
    - "!min"
    - env
    - user

  vars:

    # PorstgreSQL databases backup configuration
    backup_postgres_base_path: /var/backup/databases
    backup_postgres_databases: [ my-db, other-db ]
    
    backup_items:
 
      # Local archive backup #1
      - backup_conf_name: backup-system
        backup_directories:
          - '/etc'
          - '/usr/local/etc'
        backup_remote_base_path: /var/backup/system
        backup_mode: archive
        backup_privileged: true
        
      # Local archive backup #2
      - backup_conf_name: backup-user
        backup_directories:
          - '/home/me'
          - '/home/other'
        backup_remote_base_path: /var/backup/user
        backup_mode: archive
        backup_privileged: true        

      # Remote backup
      - backup_conf_name: backup-sync
        backup_directories:
          - '/var/backup/system'
          - '/var/backup/user'
        backup_remote_base_path: /var/backup/my-server

  tasks:

    - name: Init individual backup scripts
      ansible.builtin.include_role: { name: djuuu.backup, tasks_from: multi }
      loop: "{{ backup_items | default([]) }}"
      loop_control:
        loop_var: backup_item

    - name: Init database backup script
      ansible.builtin.include_role: { name: djuuu.backup, tasks_from: postgres }

    - name: Init backup script wrapper
      vars:
        has_postgres: "{{ backup_postgres_databases | default([]) | length > 0 }}"
        backup_conf_names: "{{
          backup_items | default([]) |
          community.general.json_query('[*].backup_conf_name')
        }}"
      when: (backup_conf_names | length > 1) or has_postgres
      ansible.builtin.include_role: { name: djuuu.backup, tasks_from: wrapper-script }
```

This will create:
- individual backup scripts:
  - `backup-system.sh`
  - `backup-user.sh`
  - `backup-sync.sh`
- a PostgreSQL database backup script:
  - `backup-postgres.sh`
- and a wrapper script running all aforementioned scripts sequentially:
  - `backup-all.sh`

License
-------

MIT / BSD
