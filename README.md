# Ansible role for Resticprofile

This role will setup [Resticprofile](https://github.com/creativeprojects/resticprofile) backups on a Debian/Ubuntu machine using a systemd service and timer.

It supports S3 backend or SFTP backend and will thus setup the SSH config and SSH private keys (see variables below).

## Installation

The playbook is installing (or upgrading):

- latest restic binary to `/usr/local/bin`
- latest resticprofile binary to `/usr/local/bin`
- the resticprofile configuration file from a template file found in `./resticprofile/{{ inventory_hostname }}/profiles.*` to `/root/resticprofile/profiles.*`
- password files that can be encrypted using ansible vault. These files are located in `./resticprofile/{{ inventory_hostname }}/keys/*`: they will be decrypted and saved to `/root/resticprofile/`.
- other files (like files needed for `--exclude-file`, `--files-from` or anything else you need) from `./resticprofile/{{ inventory_hostname }}/copy/*` to `/root/resticprofile/`

## Role Variables

### Restic installation

The role will download and install the restic binary (version `resticprofile_version`) into `resticprofile_path` if the file does not exist.

If you want to force the installation, overwrite the binary or update restic, you can run ansible with `--extra-vars resticprofile_install=true`.

### Restic configuration

- `resticprofile_user`: user to run restic as (`root`)
- `resticprofile_user_home`: home directory of the resticprofile_user (`/root`)
- `resticprofile_password`: password used for repository encryption
- `resticprofile_repository_name`: the name of the repository (`restic`)
- `resticprofile_check`: run `restic check` as `ExecStartPre` if true (`false`)
- `resticprofile_default_folders`: a default list of folders that restic will backup (`/etc/`, `/root` and `/var/log`)
- `resticprofile_folders`: the list of folder you want to backup
- `resticprofile_dump_compression_enabled`: enable piping to pigz for database dumps

Each folder has a `path` and an `exclude` property (which defaults to nothing). The `exclude` property is the literal argument passed to restic (exemple: `--exclude .cache --exclude .local`).

`resticprofile_default_folders` and `resticprofile_folders` are combined to form the final list of backuped folders.

- `resticprofile_databases`: a list of databases to dump

Each database has a `name` property which will be the name of the restic snapshot (`{{ database.name }}.sql`). They also have a `dump_command` property which is the command to dump the database to stdout (like `mysqldump dbname`).

- `resticprofile_forget`: run `restic forget` as `ExecStartPost` with `--keep-within {{ resticprofile_forget_keep_within }}` (`true`)
- `resticprofile_forget_keep_within`: period of time to use with `--keep-within` (`30d`)
- `resticprofile_prune`: run `restic prune` as `ExecStartPost` (`true`)

### SSH/SFTP backend configuration

The SSH configuration will be written in `{{ resticprofile_user_home }}/.ssh/config`.

- `resticprofile_ssh_host`: backend name and SSH alias for the backup host
- `resticprofile_ssh_user`: user for SSH connection
- `resticprofile_ssh_hostname`: actual SSH hostname of the backup machine
- `resticprofile_ssh_private_key`: private SSH key used to connect to the backup host
- `resticprofile_ssh_private_key_path`: path of the private key to use (`~/.ssh/backup`)
- `resticprofile_ssh_port`: SSH port to use with the backup machine (`23`)

### S3 backend configuration

- `resticprofile_ssh_enabled`: set to false
- `resticprofile_repository`: set to s3 endpoint + bucket, restic syntax (e.g. `s3:https://s3.fr-par.scw.cloud/restic-bucket`)
- `resticprofile_aws_access_key_id`: `AWS_ACCESS_KEY_ID`
- `resticprofile_aws_secret_access_key`: `AWS_SECRET_ACCESS_KEY`

### Resticprofile configuration

- `resticprofile_profiles`: Dictionary of resticprofile profiles. Each key is the profile name, and the value is the profile configuration as a YAML structure. The configuration will be written to `{{ resticprofile_user_home }}/resticprofile/profiles.yaml`.

Example:
```yaml
resticprofile_profiles:
  default:
    repository: "local:/backup"
    password-file: "password.txt"
    backup:
      verbose: true
      source:
        - "/home"
      schedule: "12:30"
  another_profile:
    repository: "sftp:backup:/backup"
    password-file: "password.txt"
    backup:
      source:
        - "/var/www"
        - "/etc"
```

See the [resticprofile documentation](https://creativeprojects.github.io/resticprofile/) for the full configuration options.

### Sytemd service and timer

A `restic-backup.service` service will be created with all the parameters defined above. The service is of type `oneshot` and will be triggered periodically with `restic-backup.timer`.

The timer is configurable as follows:

- `resticprofile_systemd_timer_on_calender`: defines the `OnCalendar` directive (`*-*-* 03:00:00`)
- `resticprofile_systemd_timer_randomized_delay_sec`: Delay the timer by a random amount of time between 0 and the specified time value. (`0`)

See the [systemd.timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) documentation for more information.

You can see the logs of the backup with `journalctl`. (`journalctl -xefu restic-backup`).

## Example playbook

```yaml
---

- hosts: myhost
  roles: resticprofile
  vars:
    resticprofile_ssh_user: backupuser
    resticprofile_ssh_hostname: storage-server.infra.tld
    resticprofile_folders:
      - {path: "/srv"}
      - {path: "/var/www"}
    resticprofile_databases:
    - {name: website, dump_command: sudo -Hiu postgres pg_dump -Fc website}
    - {name: website2, dump_command: mysqldump website2}
    resticprofile_password: mysuperduperpassword
    resticprofile_ssh_private_key: |-
      -----BEGIN OPENSSH PRIVATE KEY-----
      b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
      QyNTUxOQAAACAocs5g1I4kFQ1HH/YZiVU+zLhRDu4tfzZ9CmFAfKhL2AAAAJi02XEwtNlx
      MAAAAAtzc2gtZWQyNTUxOQAAACAocs5g1I4kFQ1HH/YZiVU+zLhRDu4tfzZ9CmFAfKhL2A
      AAAEADZf2Pv4G74x+iNtuwSV/ItnR3YQJ/KUaNTH19umA/tChyzmDUjiQVDUcf9hmJVT7M
      uFEO7i1/Nn0KYUB8qEvYAAAAE3N0YW5pc2xhc0BtYnAubG9jYWwBAg==
      -----END OPENSSH PRIVATE KEY-----
```

S3 example:

```yaml
---

- hosts: myhost
  roles: resticprofile
  vars:
    resticprofile_ssh_enabled: false
    resticprofile_repository: "s3:https://s3.fr-par.scw.cloud/restic-bucket"
    resticprofile_aws_access_key_id: xxxxx
    resticprofile_aws_secret_access_key: xxxxx
    resticprofile_folders:
      - {path: "/srv"}
      - {path: "/var/www"}
    resticprofile_databases:
    - {name: website, dump_command: sudo -Hiu postgres pg_dump -Fc website}
    - {name: website2, dump_command: mysqldump website2}
    resticprofile_password: mysuperduperpassword
```

Of course, `resticprofile_password` and `resticprofile_ssh_private_key` should be stored using ansible-vault.

## License

MIT

## Author Information

See my other Ansible roles at [angristan/ansible-roles](https://github.com/angristan/ansible-roles).
