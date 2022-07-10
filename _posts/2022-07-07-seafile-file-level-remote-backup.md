---
layout: post
title: "File-based remote backups of Seafile user data using seaf-fuse and Rclone"
tags:
- Homelab
- Selfhosting
- Backups
thumbnail_path: "blog/seafile-rclone-backup.png"
---

Over the years, I have tried out many different self-hosted file-sync solutions such as [Nextcloud](https://nextcloud.com/) or [Seafile CE](https://www.seafile.com/en/home/).
Personally, I've found the latter to be more performant and reliable which could be due to its mechanism of splitting files into chunks for synchronization vs. WebDav-based upload and download as is utilized under the hood by the Nextcloud Desktop Client.

One of the major caveats - Seafile not being Open Source - was fixed at least for the Community Edition.
Another big drawback for some folks is the fact that Seafile does not store user files as regular files on the server's file system.
In the case of e.g., Nextcloud, all user files are accessible in the same directory structure as they were uploaded.
This enables easy integration of these files into other tools such as selfhosted photo libraries, media servers.
It's as simple as pointing those other services to the `nextcloud-data` directory (for example through a read-only docker bind mount) and making sure that permissions are set correctly.
Seafile however stores all user uploaded data in a directory structure that looks as follows:

```
root@seafile:/seafile-data# ls -l storage/
total 26
drwxr-xr-x 17 seafile seafile 17 May 21 07:04 blocks
drwxr-xr-x 18 seafile seafile 18 Jul  8 11:08 commits
drwxr-xr-x 18 seafile seafile 18 Jul  8 11:08 fs
root@seafile:/seafile-data# ls -l storage/blocks/
total 120
drwxrwxr-x 258 seafile seafile 258 Apr 10 10:57 0fe88922-c364-472a-8798-a45829f29c58
drwxrwxr-x 181 seafile seafile 181 May  4 06:25 3466764c-c418-4fd8-822c-32cee7a20062
drwxrwxr-x  68 seafile seafile  68 Mar 18 12:27 46bf4455-9695-4ee7-8e66-cf6163708343
drwxrwxr-x 258 seafile seafile 258 May 21 07:11 5000c2fb-7a55-46e1-9cee-1e64f0f9c8e1
drwxrwxr-x 258 seafile seafile 258 Apr  7 18:43 5292df96-0963-4010-aeb0-c2344235a0eb
drwxrwxr-x 258 seafile seafile 258 Mar 18 12:25 5751a373-1e86-45d0-a7f8-bb6e9bea5c56
drwxrwxr-x 253 seafile seafile 253 Apr 28 00:16 583ad6a9-0203-41f6-92eb-4e6c008b4117
drwxrwxr-x 258 seafile seafile 258 Apr  8 14:41 5aa3d420-cb98-4b1b-b1be-b827c7c527e4
drwxrwxr-x 257 seafile seafile 257 Mar 18 12:37 855def1e-7ed7-4cde-9cf6-c60098d6a18d
drwxrwxr-x 226 seafile seafile 226 Jul  9 12:17 c854abb4-303b-4ea4-8254-09959dfed238
drwxrwxr-x 245 seafile seafile 245 Apr 10 10:48 c87a4b83-97c6-48eb-adf5-fd5c6032b2a8
drwxr-xr-x 258 seafile seafile 258 Mar 18 12:52 d0d8b21b-93da-4ce0-a9c5-dc7473f173f9
drwxrwxr-x 258 seafile seafile 258 Apr 10 09:08 de81f928-021d-4284-9dee-da73d722aa7c
drwxrwxr-x 110 seafile seafile 110 May 31 16:06 e5730d14-7104-4bdb-b1d4-2550a658acc1
drwxr-xr-x   3 seafile seafile   3 Mar 18 11:35 f47f72f8-f691-49d0-bca9-3c97bbb32a4f

```

This structure enables Seafiles high-performance chunk-based file syncing and nice features such as the integrated file versioning, but obviously hinders use-cases as described above since other applications do not understand Seafile's internal storage mechanism.
Since I generally prefer to have file-based backups that I can restore to a plain filesystem without relying on any third-party tools, I have pieced together a simple setup of doing this while still enjoying the benefits of Seafile in day-to-day use that I will share here.

## Getting direct file access using seaf-fuse

The solution lies in [seaf-fuse](https://manual.seafile.com/extension/fuse/), which is a FUSE driver for Seafile's storage system and provides a mount point that enables us to browse user files directly on the server.
The utility is available as a bash script in Seafile Server's installation directory, in my case `/opt/seafile/seafile-server-9.0.5/seaf-fuse.sh`.
*This is on a bare-metal installation, I will add information for docker installations in the future.*
We create a mount point and call this script as follows:

```
mkdir /mnt/seaf-fuse
seaf-fuse.sh start /mnt/seaf-fuse
```

... and are now able to access user files from the command line.

## Remote Backups using rclone
`rclone` is an excellent tool for transferring and syncing backups to local or remote places.
It comes with support for many cloud storage providers out of the box, such as Dropbox, Google Drive or object storage such as Amazon S3 or Backblaze B2.
Install rclone from your OS's package repository, call `rclone config`([Docs](https://rclone.org/docs/)) and follow the instructions to create a *remote* as the destination for our Seafile backups.
In my case, I use a Backblaze B2 bucket as the backup target and have called the remote `b2`.
I have written a very basic bash script which checks whether the seaf-fuse mount is up, mounts it if not (e.g., after a reboot) and calls `rclone sync` with a couple of options which enhance upload performance by minimizing API requests and uploading multiple files in parallel.
You will also find this script on my [GitHub](https://gist.github.com/ralphscheu/1cd684dcad8ce4368967812bdf0734bc).

```
#!/usr/bin/env bash

if [ -z "$(ls -A /mnt/seafile-fuse)" ]; then
   echo "Mountpoint /mnt/seafile-fuse empty, mounting Seafile using seaf-fuse.sh..."
   echo "/opt/seafile/seafile-server-latest/seaf-fuse.sh start /mnt/seafile-fuse"
   /opt/seafile/seafile-server-latest/seaf-fuse.sh start /mnt/seafile-fuse
fi

if [ -z "$(ls -A /mnt/seafile-fuse)" ]; then
   echo "Failed to mount Seafile: /mnt/seafile-fuse still empty! Aborting..."
else
   echo "Starting rclone backup..."
   rclone sync --transfers 8 --fast-list /mnt/seafile-fuse/ b2:seafile-fuse --verbose --log-file /root/b2_backup.log
fi
```

This script is run as a cronjob every night like so:

`30 4 * * * bash /root/b2_backup.sh`

While this is a very basic setup (no versioning of backups, backup job is run as root which is not ideal), it has served me well for the last 2 years and might serve as a starting point for more advanced setups.

Let me know if you have any questions or feedback! :)

{% include figure.html path=page.thumbnail_path caption=page.title url=page.external_url %}

