The [Restic Tools](https://github.com/WiseToad/restic-tools), a simple, but still useful wrapper scripts for [restic](https://restic.net/) backup program, that alongside with systemd capabilities helps me to setup complete backup system for all my hosts.

## PREREQUISITES

- install [mail-alert](https://github.com/WiseToad/mail-alert) to enable email alerting on systemd task failures

- install `restic` and `pwgen` for repository password generation:  

Debian:
```sh
sudo apt install -y restic pwgen
```
Fedora:
```sh
sudo dnf install -y restic pwgen
```

If the version of `restic` installed via package manager is outdated, install official binary from [GitHub](https://github.com/restic/restic/releases/):
```sh
mkdir ~/restic
cd ~/restic

wget https://github.com/restic/restic/releases/latest/download/restic_{{VERSION}}_linux_{{ARCH}}.bz2
```
Where:  
`{{VERSION}}` is the latest version of restic, e.g. `0.18.1`  
`{{ARCH}}` is the CPU architecture of your system, e.g. `amd64`

```sh
sudo mkdir -p /opt/restic-tools/bin
bzip2 -dck restic_{{VERSION}}_linux_{{ARCH}}.bz2 | sudo tee /opt/restic-tools/bin/restic >/dev/null
sudo chmod 755 /opt/restic-tools/bin/restic

sudo ln -s /opt/restic-tools/bin/restic /usr/local/bin
```

## INSTALL

```sh
mkdir -p ~/restic-tools
cd ~/restic-tools

wget https://github.com/WiseToad/restic-tools/releases/latest/download/restic-tools.tar.gz

sudo mkdir -p /opt/restic-tools
sudo tar xzf restic-tools.tar.gz -C /opt/restic-tools

sudo ln -s /opt/restic-tools/bin/restic-repo /usr/local/bin
```

## CONFIGURE

### Configure a new S3 storage

- create an S3 backet somewhere and obtain it's access key and secret key

- create directory for repository configs:
```sh
sudo mkdir -p /opt/restic-tools/config/repository
cd /opt/restic-tools/config/repository
```

- create an env file for S3 storage:
```sh
sudo mcedit {{STORAGE}}-s3.env
```
```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=... # optional
```
```sh
sudo chmod 640 {{STORAGE}}-s3.env
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
RESTIC_PASSWORD_FILE="${REPO_DIR}/repository.key"
```
Where:  
`{{REPO}}` is the name of repository  
`{{LOCATION}}` - possible values see here: [Preparing a new repository](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)

- generate password file:  
  NOTE: If repository is shared and already initialized for another host, you should copy the key from another host to this host instead of generating a new one.
```sh
pwgen -s 64 1 | sudo tee repository.key >/dev/null
sudo chmod 640 repository.key
```

- if repository will be stored in S3, add link to it's env file:
```sh
sudo ln -s ../{{STORAGE}}-s3.env
```
Where:  
`{{STORAGE}}` is the name of S3 storage

- init repository:  
  NOTE: If repository is shared and already initialized for another host, you should skip this step.
```sh
sudo restic-repo {{REPO}} init
```

### Configure a new task

Tasks are created for domains. Domain is a realm  to be backed up. It may be an app, or some part of system config. You can configure multiple tasks for a domain.

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
# this file will be sourced upon task execution
# current directory will be changed to the location of this file
# below is the example of some common approach:
restic-repo {{REPO}} backup \
    --no-scan --skip-if-unchanged --group-by host,tags \
    --tag files ... \
|| die

...
```
Where:  
`{{TASK}}` is the task name, e.g. `backup`  
`{{REPO}}` is the name of repository, configured earlier

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
  TIP:  it's especially useful if you spread backup into multiple repos
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

## REFERENCES

- [Restic Tools](https://github.com/WiseToad/restic-tools) - This project on GitHub  
- [restic](https://restic.net/) - Restic site
