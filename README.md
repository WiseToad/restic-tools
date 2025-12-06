The Restic Tools, an extremely tiny, but still useful wrapper scripts for [restic](https://restic.net/) backup program, that alongside with systemd capabilities helps me to setup complete backup system for all my hosts.

## PREREQUISITES

`Restic` should be installed first.  
If you plan to configire new repositories, `pwgen` should be installed as well.

Debian:
```sh
sudo apt install -y restic pwgen
```
Fedora:
```sh
sudo dnf install -y restic pwgen
```

TODO: add link to alerting service installation instructions

## INSTALL

```sh
sudo mkdir -p /opt/restic-tools
```

TODO: describe deployment instructions here

```sh
sudo ln -s /opt/restic-tools/bin/restic-repo /usr/local/bin
```

## CONFIGURE

### Configure a new S3 storage

- create an S3 backet somewhere and obtain it's access key and secret key

- create an env file for S3 storage:
```sh
sudo mcedit /opt/restic-tools/config/repository/{{STORAGE}}-s3.env
```
```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=... # optional
```
```sh
sudo chmod 600 /opt/restic-tools/config/repository/{{STORAGE}}-s3.env
```
Where:  
`{{STORAGE}}` is the name of S3 storage  

### Configure a new repository

- create directory for repository configs:
```sh
sudo mkdir -p /opt/restic-tools/config/repository/{{REPO}}
cd /opt/restic-tools/config/repository/{{REPO}}
```

- create env file:
```sh
sudo mcedit repository.env
```
```
RESTIC_REPOSITORY={{LOCATION}}
RESTIC_PASSWORD_FILE="${REPO_DIR}/{{REPO}}/repository.key"
```
Where:  
`{{REPO}}` is the name of repository  
`{{LOCATION}}` - possible values see here: [Preparing a new repository](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)

- generate password file:
```sh
pwgen -s 64 1 | sudo tee repository.key >/dev/null
sudo chmod 600 repository.key
```

- if repository will be stored in S3, add link to it's env file:
```sh
sudo ln -s ../{{STORAGE}}-s3.env
```
Where:  
`{{STORAGE}}` is the name of S3 storage

- init repository:
```sh
sudo restic-repo {{REPO}} init
```

### Configure a new task

Tasks are created for domains. Domain is a realm  to be backed up. It may be an app, or some part of system config. For a domain can be configured multiple tasks.

- create domain directory:
```sh
sudo mkdir -p /opt/restic-tools/config/task/{{DOMAIN}}
cd /opt/restic-tools/config/task/{{DOMAIN}}
```
Where:  
`{{DOMAIN}}` is the domain name

- create task:
```sh
sudo mcedit {{TASK}}
```
```sh
# code here will be sourced during task execution
# upon execution, current directory is set to location of this script
# below is the example of some common approach:
restic-repo {{REPO}} backup \
    --no-scan --skip-if-unchanged --group-by host,tags \
    --tag files ... \
|| die

...
```
Where:  
`{{TASK}}` - task name, e.g. `backup`  
`{{REPO}}` - name of repository, configured earlier

See also: [Backing up](https://restic.readthedocs.io/en/stable/040_backup.html)  
See also: [Manual](https://restic.readthedocs.io/en/stable/manual_rest.html)

- configure systemd service:
```sh
sudo mcedit /etc/systemd/system/{{TASK}}-{{DOMAIN}}.service
```
```
[Unit]
Description={{TASK}} {{DOMAIN})
OnFailure=alert@.service

[Service]
Type=oneshot
ExecStart=/opt/restic-tools/bin/restic-task {{TASK}} {{DOMAIN}}
```

- reload and test new service:
```
sudo systemctl daemon-reload
sudo systemctl start {{TASK}}-{{DOMAIN}}
sudo journalctl -u {{TASK}}-{{DOMAIN}}
```
IMPORTANT! Do not enable this service, since it's intended to be triggered by timer or another service, not upon system startup.

- configire timer:
```sh
sudo mcedit /etc/systemd/system/{{TASK}}-{{DOMAIN}}.timer
```	
```
[Unit]
Description=Timer for {{TASK}} {{DOMAIN}}

[Timer]
Unit={{TASK}}-{{DOMAIN}}.service
OnCalendar={{TIMESPEC}}
RandomizedDelaySec=15m
Persistent=true	

[Install]
WantedBy=timers.target
```
Where:  
`{{TIMESPEC}}` is the time specification (e.g. `04:00`), for details see: [Calendar Events](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html#Calendar%20Events)

```sh
sudo systemctl daemon-reload
sudo systemctl start {{TASK}}-{{DOMAIN}}.timer
sudo systemctl enable {{TASK}}-{{DOMAIN}}.timer
```

- OR, configure another service as the trigger:
```sh
sudo mcedit /usr/lib/systemd/system/{{PARENT}}.service
```
```
[Unit]
...
Wants={{THIS}}.service
Before={{THIS}}.service
```
Where:  
`{{PARENT}}` is the name of the trigger service  
`{{THIS}}` is the name of the service to be triggered

```sh
sudo systemctl daemon-reload
```

## USAGE

- start backup now:
```sh
sudo systemctl start backup-{{DOMAIN}}
```

- list snapshots:
```sh
sudo restic-repo {{REPO}} snapshots
```
See also: [Listing all snapshots](https://restic.readthedocs.io/en/stable/045_working_with_repos.html#listing-all-snapshots)

- list files within snapshot:
```sh
sudo restic-repo {{REPO}} ls {{SNAPSHOT}}
# {{SNAPSHOT}} may be "latest"
```
See also: [Listing files in a snapshot](https://restic.readthedocs.io/en/stable/045_working_with_repos.html#listing-files-in-a-snapshot)

- compare snapshots:
```sh
sudo restic-repo {{REPO}} diff {{SNAPSHOT1}} {{SNAPSHOT2}}
```
See also: [Comparing Snapshots](https://restic.readthedocs.io/en/stable/040_backup.html#comparing-snapshots)

- restore:
```sh
sudo restic-repo {{REPO}} restore {{SNAPSHOT}} --target {{DIR}}
# {{SNAPSHOT}} may be "latest"
```
See also: [Restoring from backup](https://restic.readthedocs.io/en/stable/050_restore.html)

- any restic command:
```sh
sudo restic-repo {{REPO}} command ...
```
See also: [Manual](https://restic.readthedocs.io/en/stable/manual_rest.html)  
See also: [Restic Documentation](https://restic.readthedocs.io/en/stable/)
