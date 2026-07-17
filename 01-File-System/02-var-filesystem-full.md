Scenario 2: /var Filesystem Full
================================
A production Linux server has separate partitions for /, /var, /home, and /opt.

At 3:15 AM, the monitoring team receives an alert:

/var filesystem usage has reached 100%.

Soon after:

- Applications stop writing logs.
- Package installation fails.
- systemd-journald reports write errors.
- Monitoring agents stop sending metrics.

The root (/) filesystem still has plenty of free space, but the server is experiencing application failures.

Understanding /var

The /var directory stores data that changes frequently, such as:

- Log files (/var/log)
- Package cache (/var/cache)
- Spool files (/var/spool)
- Temporary files (/var/tmp)
- Mail queues (/var/mail)
- Databases (depending on application)
- Containers (e.g., /var/lib/docker)
- Kubernetes data (e.g., /var/lib/kubelet)

A full /var partition can impact many services even if the root filesystem has free space.

Symptoms
- No space left on device
- Logging stops
- Applications crash
- Yum/DNF/APT updates fail
- Docker containers fail to start
- Kubernetes pods fail
- System monitoring stops reporting

Investigation Workflow
===================
-- Step 1: Verify /var Usage
df -h /var

Explanation

Displays usage information for the filesystem that contains /var.
| Option | Meaning               |
| ------ | --------------------- |
| `-h`   | Human-readable output |

Filesystem      Size Used Avail Use% Mounted on
/dev/sdb1       50G 50G 0G 100% /var

Interpretation
/var is mounted separately.
- Total capacity: 50 GB.
- Free space: 0 GB.
  
-- Step 2: Determine Which Directory Uses the Most Space
  
  du -xh --max-depth=1 /var | sort -hr

  Explanation
- du → Disk usage.
- -x → Stay within the same filesystem.
- -h → Human-readable.
- --max-depth=1 → Display only first-level directories.
- sort -hr → Sort from largest to smallest.

35G /var/log
8G /var/lib
4G /var/cache
2G /var/tmp
500M /var/spool

Interpretation

Most of the space is being used by /var/log

-- Step 3: Inspect Large Log Files
find /var/log -type f -size +100M -exec ls -lh {} \;
Explanation
- find → Search files.
- -type f → Files only.
- -size +100M → Larger than 100 MB.
- -exec ls -lh → Show detailed file information.

-rw-r----- 1 root root 12G messages
-rw-r----- 1 root root 8G secure
-rw-r--r-- 1 root root 5G application.log

Interpretation

Multiple log files have grown abnormally.

-- Step 4: Check if Log Rotation Is Working
logrotate -d /etc/logrotate.conf

Explanation
- -d → Debug mode (does not modify files).

This validates whether log rotation is configured correctly.

Common Problems
- Missing logrotate configuration.
- Incorrect file permissions.
- Rotation disabled.
- Application keeping log files open.

-- Step 5: Examine System Journal Usage
journalctl --disk-usage

Archived and active journals take up 15.6G on disk.

Clean Old Journals

journalctl --vacuum-time=7d

Explanation

- Retains only the last 7 days of journal logs.

Alternative:

journalctl --vacuum-size=1G

- Limits journal storage to approximately 1 GB.

-- Step 6: Check Package Cache depending on the OS type

du -sh /var/cache/dnf
du -sh /var/cache/yum
du -sh /var/cache/apt

If the cache is consuming significant space:
dnf clean all
apt clean

-- Step 6: Look for Deleted Files Still Held Open
lsof +L1

java  1256 appuser 10w REG 253,1 15G /var/log/app.log (deleted)

Interpretation

Although the file has been deleted, the Java process still has it open, so the disk space is not released until the process is restarted.

Root Cause
=========

A Java application was configured with DEBUG logging, producing several gigabytes of logs every hour. At the same time, logrotate failed because of an incorrect configuration file, preventing old logs from being rotated.

Resolution
============
- Compress or archive old logs.
- Delete obsolete logs after verifying they are no longer needed.
- Vacuum old systemd journals.
- Clean package manager caches.
- Restart applications holding deleted log files open.
- Fix the logrotate configuration.
- Reduce application logging from DEBUG to INFO in production.

Prevention
============
- Enable monitoring and alerts when /var exceeds 80% usage.
- Verify logrotate runs successfully.
- Configure SystemMaxUse for journald.
- Schedule periodic cache cleanup.

Avoid long-term DEBUG logging in production.