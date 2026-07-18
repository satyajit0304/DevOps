Scenario 4: Deleted File Still Consuming Disk Space
==================================================

A production Linux server reports that the root filesystem is 100% full. An administrator deletes a 20 GB log file, but running df -h still shows the filesystem at 100%.

The application continues to fail with:

No space left on device

However, when checking the directory, the large log file is gone.

- Concept
=============
Deleting a file in Linux does not always free disk space immediately.
A file's disk space is released only when:

- The file has been deleted (directory entry removed), and
- No running process has the file open.

If a process still has the file open, the inode and data blocks remain allocated until the process closes the file or exits.

How Linux Handles File Deletion
==================================
Suppose a Java application writes to:

/var/log/application.log

The application opens the file and receives a file descriptor.

Java Process
      │
      ▼
File Descriptor (FD 15)
      │
      ▼
Inode
      │
      ▼
Disk Blocks

If someone runs:

rm /var/log/application.log

Linux removes only the directory entry.

The Java process still owns the open file descriptor.

Java Process
      │
      ▼
FD 15
      │
      ▼
Inode
      │
      ▼
20 GB Disk Blocks

The file is invisible in the filesystem, but the space remains allocated.

Symptoms
==============
- df -h shows 100% usage.
- du -sh / reports much less usage.
- Large log file is missing.
- Application still running normally.
- Restarting the application suddenly frees disk space.

Investigation Workflow
============================

- Step 1: Verify Filesystem Usage
====================================
df -h
Example Output
Filesystem      Size Used Avail Use%
/dev/sda2       100G 100G 0G 100%
Interpretation

The filesystem is completely full.

- Step 2: Compare with Directory Usage
=======================================
du -sh /

Example

72G /

Observation

- Filesystem reports 100 GB used
- Directory scan reports 72 GB used

Approximately 28 GB is missing.

This discrepancy strongly suggests deleted-but-open files.

- Step 3: Search for Deleted Open Files
============================================
lsof +L1


Option	Meaning
- lsof	List Open Files
- +L1	Show files with link count less than 1 (deleted files)

Example Output
COMMAND   PID USER   FD TYPE DEVICE SIZE/OFF NLINK NAME
java     4289 app   12w REG 253,2 20G      0 /var/log/app.log (deleted)

Column	           Meaning
COMMAND	           Process   name
PID	               Process ID
USER	           Process owner
FD	               File descriptor
TYPE	           File type
SIZE/OFF	       File size
NLINK	           Number of directory links
NAME	           Deleted filename

Notice

NLINK = 0

This means

- directory entry removed
- inode still exists
- process still owns it

-Step 4: Verify Process
=======================
ps -fp 4289

Example

UID PID PPID CMD

app 4289 1 java -jar payment-service.jar

The Java application is still running.

- Step 5: Check File Descriptor
========================================
ls -l /proc/4289/fd

Example

12 -> /var/log/app.log (deleted)

Linux exposes every open file descriptor through

/proc/<PID>/fd/

This confirms the process still owns the deleted file.

- Step 6: Check Process Status
===============================
cat /proc/4289/status

Useful information

- Process state
- Memory usage
- Open threads
- File descriptor limits

- Step 7: Count Open File Descriptors
==================================
ls /proc/4289/fd | wc -l

Example

356

Applications with many open descriptors may delay resource cleanup.

- Step 8: Decide the Recovery Method
======================================
- Option A (Preferred)

Restart the application gracefully.

Example

systemctl restart payment.service

Why?

Restarting closes all file descriptors.

Kernel releases

inode
disk blocks

Immediately.

- Option B

Reload the application if supported.

Example

systemctl reload nginx

Some services reopen log files without a full restart.

- Option C

Kill the process (last resort).
kill -15 4289

If the process ignores SIGTERM
kill -9 4289


- Step 9: Verify Space Released
================================
df -h

Example

Filesystem      Size Used Avail Use%
/dev/sda2       100G 78G 22G 78%

Disk space has been released.

- Step 10: Verify No Deleted Files Remain
========================================
lsof +L1

Output

(no output)

Investigation complete.

Why rm Doesn't Immediately Free Space

Many people believe
rm deletes a file

Actually,
rm removes only the directory entry.
The inode remains until
Reference Count = 0
AND
No process has the inode open.

- Root Cause
=========================
A Java application generated a 20 GB log.
An administrator deleted application.log without restarting the application.

The process continued writing to the deleted inode.

- Resolution
=====================
- Identify deleted files using
lsof +L1
- Restart affected application
- Verify disk usage
- Reconfigure log rotation
