# Anarcho-Tech NYC: Duplicity [![Build Status](https://travis-ci.org/AnarchoTechNYC/ansible-role-duplicity.svg?branch=master)](https://travis-ci.org/AnarchoTechNYC/ansible-role-duplicity)

An [Ansible role](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) for installing the [Duplicity](https://nongnu.org/duplicity/) backup software and managing simple backup jobs. Notably, this role has been tested with [Raspbian](https://www.raspbian.org/) on [Raspberry Pi](https://www.raspberrypi.org/) hardware. This role's purpose is to make it simple to create and manage backup jobs for simple servers.

# About Duplicity backup jobs

Duplicity is a space- and bandwidth-efficient backup software utility built atop the [rsync](https://rsync.samba.org/) and [GnuPG](https://gnupg.org/) technologies. It provides a relatively simple way to create full or incremental backups, either from a command line or using a variety of graphical user interfaces. By default, it uses password-based [(symmetric) encryption](https://simple.wikipedia.org/wiki/Symmetric-key_algorithm) to protect [data at rest](https://en.wikipedia.org/wiki/Data_at_rest), and is compatible with numerous storage backends including [Free Software](https://www.gnu.org/philosophy/free-sw.html) and even technocapitalist SAASS offerings.

Each backup job is one or more invocations of `duplicity(1)` with options that can be configured on a per-job basis. The initial backup job is usually a full backup, with subsequent backup jobs performing some number of incremental backups until a newer full backup is performed again. Optionally, these backup jobs are followed up with a job that removes very old backups from the backup target. By crafting a schedule of full, incremental, and removal jobs, a custom backup retention policy can be implemented in a flexible manner.

# Configuring backup jobs

Each backup job is one or more invocations of `duplicity(1)` with options that can be configured on a per-job basis. To configure these invocations, use the `duplicity_backup_jobs` list. Each item in the list is a dictionary that describes how and when to perform the backup job. A job description can include the following keys:

* `name`: The name of the backup job. Backup scripts with this name as a prefix will be generated. This key is required.
* `state`: Whether the backup job should be scheduled (`present`) or be removed from the managed node (`absent`). The default is `present`.
* `user`: The operating system user account that will be performing the backup job. If this is omitted, the `duplicity_default_backup_user` will be used.
* `source_directory`: The directory to make a backup copy of. The specific files that are backed up will depend on any `--exclude` and `--include` options specified in the `options` list, see below.
* `target_url`: The Duplicity backup target in URL format. See the *URL FORMAT* section in `duplicity(1)` for more information.
* `options`: List of runtime options to pass to Duplicity.
* `env`: Dictionary of environment variables to set for the backup job invocation. Useful variables are, for example, `RSYNC_CONNECT_PROG` or `PASSPHRASE`. Refer to `duplicity(1)` for more information.
* `ssh_known_host`: If your backup job relies on an SSH-backed target, use this key to preload the backup user's `~/.ssh/known_hosts` file with the correct host key. This key is a dictionary with two subkeys:
    * `name`: Name of the SSH server host.
    * `key`: Public host key of the SSH server, in `known_hosts` file format (i.e., with the `name` value prepended).
* `cron_schedule`: Scheduling parameters for the backup invocation's cron job. This is a dictionary with the following subkeys that correspond to the scheduling fields in a `crontab(5)` file:
    * `minute`
    * `hour`
    * `day`
    * `month`
    * `weekday`
* `follow_up`: Each Duplicity backup job can be associated with a "follow up" job whose purpose is to clean/rotate/remove very old backups off the backup target. This dictionary describes this follow-up job.
    * `duplicity_action`: The Duplicity action to follow up backup jobs with. See the "Actions" section in `duplicity(1)` for more info.
    * `duplicity_action_arg`: The argument to pass to the Duplicity action.
    * `schedule`: The schedule for the cron job to execute the backup follow-up job on. This is a dictionary with the same format as the `cron_schedule` key, listed above.

Some examples may prove illustrative:

1. Have Alice backup their own home folder once every week to a backup target via rsync over an SSH connection.
    ```yaml
    duplicity_backup_jobs:
      - name: "Backup Alice's home folder"
        user: alice
        source_directory: "{{ '~/' | expanduser }}"
        target_url: "rsync://alice@example.invalid/backups"
        options:
          - "--full-if-older-than 2M"
        env:
          PASSPHRASE: "$(/bin/cat ~/.duplicity.secret)"
        ssh_known_host:
          name: "example.invalid"
          key: "example.invalid ssh-rsa <… SSH HOST'S PUBLIC RSA KEY HERE …>"
        cron_schedule:
          minute: 0
          hour: 0
          day: "*"
          month: "*"
          weekday: 0
    ```
    The above configuration will create a backup script, `~alice/bin/duplicity_jobs/Backup-Alice's-home-folder.sh`, that will invoke `duplicity` in the following manner:
    ```sh
    PASSPHRASE=$(/bin/cat ~/.duplicity.secret) duplicity --full-if-older-than 2M /home/alice rsync://alice@example.invalid/backups
    ```
    A cron job will be created in `alice`'s `crontab` file that looks like this:
    ```
    #Ansible: Backup Alice's home folder
    0 0 * * 0 ~alice/bin/duplicity_jobs/Backup-Alice's-home-folder.sh >/dev/null 2>&1
    ```
    Finally, Alice's `~alice/.ssh/known_hosts` file will be set up to validate the backup target's SSH server using the SSH host's public key for `example.invalid`.

    For this backup to succeed, you should ensure that the backup archive's symmetric key (password) is stored in the `~/.duplicity.secret` file, or else the cron job will fail as it cannot supply a password with which to encrypt (or decrypt) the Duplicity backup archive. 

See the comments in [`defaults/main.yaml`](defaults/main.yaml) for additional details.

## Additional role variables

* `duplicity_default_backup_user`: The default user account to use when a given backup job does not specify a `user` itself.
