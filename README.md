# BBackups
## Introduction
This is a simple bash implementation of the backup scheme described in the blog post [Easy Automated Snapshot-Style Backups with Linux and Rsync](http://www.mikerubel.org/computers/rsync_snapshots/), which also formed the basis of `rsnapshot`.
## Description
The `bbackups` process is designed for remote pull-style backups with `rsync` over a trusted network connection. It maintains a hot backup of the remote filesystem which we refer to as the *mirror*. It also keeps a number of hourly, daily, weekly and monthly snapshots which use hard links to track only changes to files. Additionally, this implementation makes use of systemd timers to maintain process dependencies.
### The Order of Operations
Ignoring the snapshot rotation which happens before every step, `bbackups` goes off the following order
1. If a weekly snapshot exists, and the latest monthly snapshot is older than 28 days, create a monthly snapshot
2. If a daily snapshot exists, and the latest weekly snapshot is older than 6 days, create a weekly snapshot
3. If an hourly snapshot exists, and the latest weekly snapshot is older than 23.5 hours, create a daily snapshot
4. Create an hourly snapshot from the backup mirror
5. Use `rsync` to mirror the remote files to the local mirror.
### Use of archived hardlinked copies
Without explaining too much of what Mike Rubel describes in their blog post [Easy Automated Snapshot-Style Backups with Linux and Rsync](http://www.mikerubel.org/computers/rsync_snapshots/), the snapshots, as distinct from the rsync mirror, keep a record of the files at a given time as a hard link to the mirror. This way, if a file is deleted on the remote, and this change is propagated to the backup server, the file may be deleted in the mirror but retained in the snapshot. *Additionally*, `rsync` behaves in such a way that files are unlinked if the sync captures a file *modification*, so file changes are kept intact within a snapshot as well. This saves on disk space while ensuring that the snapshot looks and feels exactly like the filesystem at the time of capture.
### Use of rsync
The backup script can be modified at will to include any `rsync` flags the user wants, however some flags are *strongly recommended*:
 - Please use an `ssh` key and restrict the use of that key on the remote system being backed up.
 - Since the snapshot system already retains files if they are deleted, it is totally safe (and will save disk space) to use the `--delete` flag!
 - The backup script comes pre-configured to keep and rotate logs in the `/var/log/backup` directory. While not necessary, it is useful to debug and make sure your backups succeed.
## Setting up ssh keys
On the remote system, the public ssh key should be added to the `~/.ssh/authorized_keys` file of whichever user is used on the remote to read and back up files. Additionally, a command string may be used to limit the command that can be run using this ssh key.
To fetch the command that is run on the remote server, run the `rsync` command with the `-v -n` flags. For more information, [Using Rsync and SSH: Keys, Validating, and Automation](https://troy.jdmz.net/rsync/index.html) is instructive.
Once you have the command, add this to the start of the public key in the `authorized_keys` file:
```
command="rsync --server ..." AAAA... user@backup.host
```
