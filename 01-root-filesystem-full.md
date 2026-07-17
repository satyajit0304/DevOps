Scenario 1: Root Filesystem Full
================================
At 2:00 AM, your monitoring system alerts that a production Linux server is at 100% disk utilization on the root (/) filesystem. Users begin reporting application failures, scheduled jobs stop working, and logins become unreliable.

-- Symptoms
===========
No space left on device
Applications fail to write logs
SSH login is slow
Cron jobs fail
Database backups fail
New files cannot be created

-- Investigation Workflow
=========================
--- Step 1: Check Filesystem Usage
===================================
df -h
What does it do?

Displays mounted filesystems and their disk usage in a human-readable format.

Option explanation
Option	Meaning
-h	Human-readable sizes (MB, GB, TB)
Example Output
Filesystem      Size Used Avail Use% Mounted on
/dev/sda2       100G 100G 0G 100% /
tmpfs            16G  20M 16G   1% /run

Interpretation
- /dev/sda2 is mounted as /
- Total size = 100 GB
- Used = 100 GB
- Free = 0 GB
- Filesystem is completely full

--- Step 2: Find Which Filesystem is Full
==========================================
  df -Th
  Filesystem Type Size Used Avail Use%
/dev/sda2 ext4 100G 100G 0G 100%
Knowing the filesystem type helps with troubleshooting and repair procedures.

--- Step 3: Identify Large Top-Level Directories
==================================================
du -xh --max-depth=1 /
Explanation
- du → Disk usage
- -x → Stay on the same filesystem (ignore mounted filesystems)
- -h → Human-readable
- --max-depth=1 → Show only immediate subdirectories
Example Output
45G /var
20G /opt
12G /home
8G /usr

Interpretation
Most disk space is under /var

--- Step 4: Investigate /var
===================================
du -xh --max-depth=1 /var
35G /var/log
8G /var/cache
1G /var/tmp
The primary consumer is /var/log

--- Step 5: Find Large Files
=============================
find /var/log -type f -size +500M -exec ls -lh {} \;

Option explanation
- -type f → Files only
- -size +500M → Larger than 500 MB
- -exec → Run a command on each result
- ls -lh → Show file details in human-readable format
-rw-r----- 1 root root 18G app.log
-rw-r----- 1 root root 6G access.log

--- Step 6: Check Systemd Journal Size
=======================================
journalctl --disk-usage
Archived and active journals take up 8.2G on disk.

If journals are consuming excessive space:
journalctl --vacuum-size=500M

This reduces journal storage to approximately 500 MB.

--- Step 7: Identify Deleted Files Still Consuming Space
========================================================
lsof +L1
Explanation
- lsof → Lists open files
- +L1 → Shows open files with a link count less than 1 (deleted but still open)

java 2456 appuser 12w REG 253,0 15G /var/log/app.log (deleted)

This indicates the process still holds the deleted file open.

--- Step 8: Check Largest Individual Files
============================================
find / -xdev -type f -exec du -h {} + | sort -hr | head -20
Breakdown
- -xdev → Stay on the current filesystem
- sort -hr → Sort by size, human-readable
- head -20 → Display the largest 20 files

--- Step 9: Verify Available Inodes
====================================
df -i

Filesystem Inodes IUsed IFree IUse%
/dev/sda2 6553600 6553600 0 100%

--- Step 10: Inspect Recently Modified Large Files
===================================================
find /var/log -type f -mtime -1 -ls

Explanation
- -mtime -1 → Modified within the last 24 hours
- -ls → Display detailed file information

Useful for identifying runaway logging.

Root Cause
===========

Application logging was accidentally set to DEBUG level, causing logs to grow from a few MB to tens of GB in a matter of hours.

Resolution
==========
- Rotate or archive old logs.
- Remove unnecessary cache files.
- Vacuum the systemd journal.
- Restart applications holding deleted files open.
- Correct the application's logging configuration.
- Verify free space with df -h.

Prevention
===============
- Configure logrotate.
- Set journal size limits (SystemMaxUse in journald.conf).
- Monitor filesystem usage with alerts (e.g., Prometheus, Azure Monitor).
Avoid DEBUG logging in production unless required.
