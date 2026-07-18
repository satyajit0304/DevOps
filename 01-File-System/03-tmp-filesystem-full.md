Scenario 3: /tmp Filesystem Full
================================

A CI/CD pipeline suddenly starts failing during deployments. Developers also report that application uploads are failing, and system administrators cannot install packages. Investigation reveals that the /tmp filesystem has reached 100% usage.

Many enterprise Linux servers mount /tmp as a separate filesystem or as a tmpfs (memory-backed filesystem) for security and performance reasons. When /tmp fills up, many applications that rely on temporary files stop functioning correctly.

Understanding /tmp
=================
The /tmp directory is used for temporary files created by:

- Applications
- Installers
- Package managers (dnf, yum, apt)
- Java applications
- Python scripts
- Shell scripts
- CI/CD tools (Jenkins, GitLab Runner, Azure DevOps agents)


By convention, files in /tmp are temporary and may be cleaned up automatically on reboot or by scheduled cleanup jobs, depending on the system configuration.

Symptoms
=========
- No space left on device
- Package installation fails
- Application uploads fail
- Temporary file creation errors
- Jenkins builds fail
- Azure DevOps agent jobs fail
- Docker image builds fail
- Java applications throw java.io.IOException

Example:

java.io.IOException: No space left on device

Investigation Workflow
=======================
- Step 1: Check /tmp Filesystem Usage
======================================
df -h /tmp
Explanation

Shows disk usage for the filesystem containing /tmp.


Option	Meaning
- -h	Human-readable output
Example Output
Filesystem      Size Used Avail Use% Mounted on
tmpfs            8G   8G     0 100% /tmp
Interpretation

The tmpfs mounted on /tmp has no available space.

- Step 2: Check Whether /tmp Is a Memory Filesystem
======================================================
mount | grep /tmp

Explanation
- mount displays mounted filesystems.
- grep /tmp filters the output for the /tmp mount.

Example Output
tmpfs on /tmp type tmpfs (rw,nosuid,nodev)
Interpretation

/tmp is a tmpfs, meaning it uses RAM (and swap if needed), not a physical disk partition.

This is important because cleaning /tmp frees memory as well as temporary storage.

- Step 3: Check Size of Directories Under /tmp
================================================

du -xh --max-depth=1 /tmp | sort -hr

Explanation
- du reports disk usage.
- -x stays on the same filesystem.
- --max-depth=1 summarizes first-level directories.
- sort -hr sorts from largest to smallest.

Example Output
6.5G /tmp/jenkins
900M /tmp/tomcat
250M /tmp/upload
Interpretation

The Jenkins workspace is consuming most of the space.

- Step 4: Find Large Files
=============================
find /tmp -type f -size +100M -exec ls -lh {} \;

Explanation
- -type f searches only regular files.
- -size +100M finds files larger than 100 MB.
- -exec ls -lh displays detailed information.

Example Output
-rw------- 1 jenkins jenkins 2.8G build.tar
-rw-r--r-- 1 app app 1.2G upload.tmp

Interpretation

Large temporary files were not cleaned up.

- Step 5: Check Which Users Own the Files
============================================
find /tmp -printf "%u\n" | sort | uniq -c

Explanation
- %u prints the file owner.
- sort groups owners.
- uniq -c counts occurrences.

Example Output
2400 jenkins
800 oracle
120 appuser

Interpretation

Most temporary files belong to the jenkins user.

- Step 6: Find Old Temporary Files
===================================
find /tmp -type f -mtime +7

Explanation
- -mtime +7 finds files modified more than seven days ago.

Example Output
/tmp/build123.tar
/tmp/archive.log
Interpretation

These stale files are good candidates for cleanup after verifying they are no longer needed.

- Step 7: Check Open Files in /tmp
====================================

lsof +D /tmp

Explanation
- lsof lists open files.
- +D /tmp recursively checks files currently open under /tmp.

Example Output
java  4218 appuser 18u REG /tmp/upload.tmp
Interpretation

The Java process is actively using the file. Deleting it before the process finishes could cause application errors.

- Step 8: Check Available Memory (for tmpfs)
==========================================
free -h
Explanation

Displays RAM and swap usage.

Example Output
              total   used   free  shared  buff/cache available
Mem:           16Gi    14Gi   500Mi   2Gi      1.5Gi      800Mi
Swap:           4Gi   3.5Gi   500Mi

Interpretation

Because /tmp is a tmpfs, high memory usage can contribute to /tmp filling up.

- Step 9: Remove Old Temporary Files
===================================

Only after confirming they are safe to delete:

find /tmp -type f -mtime +7 -delete

Explanation

Deletes files older than seven days.

Warning: Never blindly delete files from /tmp on a production system without confirming they are not in use.

- Step 10: Verify Cleanup
===============================
df -h /tmp
Example Output
Filesystem      Size Used Avail Use% Mounted on
tmpfs            8G 2.1G 5.9G 27% /tmp
Interpretation

The cleanup was successful, and sufficient free space is available.

Root Cause
=================

A Jenkins pipeline generated multi-gigabyte build artifacts in /tmp and failed to remove them after completion. Over several days, these files accumulated until the tmpfs was exhausted.

Resolution
===========
- Identify the largest consumers of /tmp.
- Verify files are no longer in use.
- Remove stale temporary files.
- Restart any applications holding deleted files open if necessary.
- Update CI/CD jobs to clean up temporary artifacts after successful builds.

Prevention
============
- Configure scheduled cleanup of /tmp using systemd-tmpfiles or tmpwatch.
- Ensure CI/CD pipelines remove temporary files.
- Monitor /tmp usage with alerts.
- Avoid storing large application data in /tmp; use dedicated storage locations instead.
- Review tmpfs size if workloads legitimately require more temporary space.