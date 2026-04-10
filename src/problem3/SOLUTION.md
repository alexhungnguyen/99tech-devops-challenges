## Troubleshooting steps

Get a picture of what's consuming the disk

### 1. Switch to super user to have unrestricted access to all directories if possible
```bash
sudo su
```

### 2. Check overall disk usage and block device layout
```bash
df -h
lsblk
```

`df -h` shows filesystem-level usage (how full each mounted filesystem is), while `lsblk` shows the block device hierarchy (disks, partitions, and their sizes) - useful for identifying unpartitioned space or unmounted volumes.

Example output
```bash
root@ip-172-31-47-247:/home/ubuntu# df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/root        6.8G  2.2G  4.6G  33% /
tmpfs            456M     0  456M   0% /dev/shm
tmpfs            183M  892K  182M   1% /run
tmpfs            5.0M     0  5.0M   0% /run/lock
efivarfs         128K  3.6K  120K   3% /sys/firmware/efi/efivars
/dev/nvme0n1p16  881M   94M  726M  12% /boot
/dev/nvme0n1p15  105M  6.2M   99M   6% /boot/efi
tmpfs             92M   12K   92M   1% /run/user/1000

---

ubuntu@ip-172-31-47-247:~$ lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0          7:0    0 27.8M  1 loop /snap/amazon-ssm-agent/12322
loop1          7:1    0 48.1M  1 loop /snap/snapd/25935
loop2          7:2    0   74M  1 loop /snap/core22/2339
nvme0n1      259:0    0    8G  0 disk 
├─nvme0n1p1  259:1    0    7G  0 part /
├─nvme0n1p14 259:2    0    4M  0 part 
├─nvme0n1p15 259:3    0  106M  0 part /boot/efi
└─nvme0n1p16 259:4    0  913M  0 part /boot

```

### 3. Find the largest directories from root and order by size from largest to smallest
```bash
du -sh /* 2>/dev/null | sort -rh | head -20
```

Example output
```bash
root@ip-172-31-47-247:/home/ubuntu# du -sh /* 2>/dev/null | sort -rh | head -20
1.5G	/usr
725M	/var
508M	/snap
...
```

### 4. Review nginx config
Check nginx config for cache paths if being set
```bash
grep -r "proxy_cache_path\|proxy_temp_path\|client_body_temp_path\|fastcgi_cache_path" /etc/nginx/
```
Check nginx config for log location
```bash
grep -r "access_log\|error_log" /etc/nginx
```

Example output
```bash
root@ip-172-31-47-247:/home/ubuntu# grep -r "access_log\|error_log" /etc/nginx
/etc/nginx/nginx.conf:error_log /var/log/nginx/error.log;
/etc/nginx/nginx.conf:	access_log /var/log/nginx/access.log;
```

### 5. Inspect some potential directories
Log files often grow quickly if not managed correctly. A common location for logs is the `/var/log` directory.
```bash
du -sh /var/log/* | sort -rh | head -10
```

Check apt cache
```bash
du -sh /var/cache/apt/*
```

Example output
```bash
root@ip-172-31-47-247:/home/ubuntu# du -sh /var/log/* | sort -rh | head -10
17M	  /var/log/journal
5M	  /var/log/nginx
212K	/var/log/syslog
...

---

root@ip-172-31-47-247:/home/ubuntu# du -sh /var/cache/apt/*
133M	/var/cache/apt/archives
60M	  /var/cache/apt/pkgcache.bin
60M	  /var/cache/apt/srcpkgcache.bin
```

### 6. Find large individual files (e.g., >100MB)
```bash
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

### 7. Check for deleted files still held open by processes
```bash
lsof +L1
```

## Impacts

A full disk has a cascading failure profile that goes well beyond NGINX itself:

| Layer | Effect |
|---|---|
| **NGINX** | Fails to write logs; may reject new connections or crash workers entirely |
| **OS temp space** | `/tmp` writes fail, breaking many userspace programs that rely on temp files |
| **systemd / journald** | Cannot write journal entries; service restarts and health checks may silently fail |
| **cron / scheduled jobs** | Jobs that produce any output fail, potentially skipping backups or maintenance tasks |
| **SSH** | New SSH sessions may be refused if PAM or the shell cannot write to disk, locking you out of the VM |

The risk compounds quickly - a disk that is 99% full during off-hours can hit 100% quickly under normal traffic load, turning a warning into a full outage.

### Emergency recovery: disk is 100% full and SSH is unavailable

If SSH access is lost because the disk is completely full, the fastest recovery path on AWS is to expand the underlying EBS volume without needing to stop the instance:

1. **Resize the EBS volume** in the AWS Console (EC2 -> Volumes -> Modify Volume) or via CLI:
```bash
aws ec2 modify-volume --volume-id vol-xxxxxxxx --size <new_size_gb>
```

2. **Worst case**: After resizing the EBS volume, the filesystem doesn't automatically use the new space, it's still not possible to SSH to the VM, then I have to do follow recovery instance method, which will cause downtime on the main app
- Stop the VM
- Detach root volume
- Launch new Ubuntu 24.04 temp instance (same AZ)
- Attach original volume as `/dev/sdf`
- SSH to temp instance -> mount the original volume -> cleanup that volume
- Stop the temp instance, detach the volume
- Re-attach the original volume to to the original instance, start the instance again

3. Once SSH access is restored, immediately investigate and resolve the root cause to prevent recurrence.

## Potential scenarios, root causes

### Root cause 1: Unrotated and excessive nginx logs

#### Expected behavior:
The size of `/var/log/nginx` (or any other directory set as the nginx log file location) is high.

#### Why it happens
NGINX writes to access.log and error.log by default. As a load balancer handling significant traffic, the access log can grow very quickly without rotation.

#### Recovery steps:
- Immediate fix: Backup current log files (by upload to S3 or copy to our local machine temporarily) then truncate the active log file without restarting nginx
```bash
truncate -s 0 /var/log/nginx/access.log
```

- Verify space is reclaimed with `df -h`

- Verify if logrotate is installed. Although it's very likely that Logrotate is pre-installed on Ubuntu 24.04
```bash
dpkg -l | grep logrotate
logrotate --version
```

- Setup logrotate properly: Ensure `/etc/logrotate.d/nginx` exists with sensible settings - daily rotation, compression, a retention window (e.g., 7–14 days), and the postrotate signal so NGINX reopens file handles. Create new one with a sample configuration below if not existed yet
```
# Rotate NGINX logs daily, keep 7 days
/var/log/nginx/*.log {
    daily
    missingok
    # Keep 7 rotated logs before deletion
    rotate 7
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        # send a USR1 signal, prompting NGINX to reopen its log file handles after rotation
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

### Root cause 2: Stale files (old kernels, package cache, temp files)

#### Expected behavior:
`du -sh /var/cache/apt/*` shows several GB, `dpkg --list | grep linux-image` shows many old kernels, or `/tmp` contains large stale files.

#### Why it happens: 
Over time, apt caches downloaded `.deb` packages in `/var/cache/apt/archives/`, old kernel versions accumulate under /boot and /usr/lib/modules/, and temporary files pile up in /tmp

### Recovery steps:
- Clean the apt cache
```bash
apt-get clean          # removes all cached .deb files
apt-get autoremove     # removes orphaned dependencies
```
- Remove old kernels, keep the current and the one previous
```bash
apt-get autoremove --purge
```

- Cleanup unused temporary file in `/tmp` if there's any

### Root cause 3: Journal log growth

#### Expected behavior:
`du -sh /var/log/journal/` returns a very large number, or `journalctl --disk-usage` reports gigabytes of stored journals.

Example output
```bash
root@ip-172-31-47-247:/home/ubuntu# du -sh /var/log/journal/
17M	/var/log/journal/
root@ip-172-31-47-247:/home/ubuntu# journalctl --disk-usage
Archived and active journals take up 16.0M in the file system.
```

#### Why it happens:
On Ubuntu, `systemd-journald` stores persistent logs under `/var/log/journal/`. By default there's no aggressive size cap, and if the system has been running for months - the journal log can grow quickly.

#### Recovery steps:
- Vacuum the journal to a reasonable size:
```bash
# Deletes the oldest archived journal files until the total journal disk usage drops below 500MB
journalctl --vacuum-size=500M
```

- Set a permanent cap in `/etc/systemd/journald.conf`
```
[Journal]
SystemMaxUse=500M
SystemKeepFree=2G
```
`SystemKeepFree=2G` means it will 2GB free space on the filesystem for other uses. Journal stops growing when filesystem free space drops to 2GB, even if under 500MB limit

- After update the file, execute those commands
```bash
# Restart journald (safe, zero downtime)
sudo systemctl restart systemd-journald

# Verify active settings
systemd-analyze cat-config systemd/journald.conf | grep -E 'SystemMaxUse|SystemKeepFree'
```

### Root cause 4: Deleted files held open by running processes

#### Expected behavior:
`df -h` reports high disk usage but `du -sh /*` accounts for far less space. Running lsof +L1 reveals large deleted files still open.

#### Why it happens: 
When a log file or temp file is deleted while a process still has it open, the kernel keeps the data on disk until the file descriptor is closed. The space is not reclaimed until the process releases the handle

#### Recovery steps:
- Identify the process
```bash
lsof +L1 | grep deleted

# Example output
# COMMAND   PID     USER   FD   TYPE DEVICE   SIZE/OFF NLINK   NODE NAME
# nginx    3082     root    4w   REG  259,1          0     0 262419 /var/log/nginx/access.log~ (deleted)
# nginx    3084 www-data    4w   REG  259,1          0     0 262419 /var/log/nginx/access.log~ (deleted)
# nginx    3085 www-data    4w   REG  259,1          0     0 262419 /var/log/nginx/access.log~ (deleted)
# tail    52866     root    3r   REG  259,1 1073741824     0    503 /tmp/fake_log.txt (deleted)
```

- Restart or reload the process holding the file open, which causes the kernel to free the space immediately. If it's nginx, run this command to reload
```bash
systemctl reload nginx
```

- Verify with `df -h` that space is reclaimed
