---
tags:
  - daily-usage
  - usage
  - journal
---

# journalctl Daily Usage Guide

`journalctl` is a command-line utility for querying and displaying logs from `journald`, systemd's logging service.

## 1. Basic Log Viewing

- **View all logs**: `journalctl`
  ```shell
  journalctl
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:00:00 hostname systemd[1]: Starting Hostname Service...
  Mar 27 10:00:00 hostname systemd[1]: Reached target Local File Systems.
  Mar 27 10:00:00 hostname kernel: Linux version 5.4.0-77-generic (buildd@lcy02-amd64-CI-build)
  ...
  ```

- **View logs in reverse chronological order (newest first)**: `journalctl -r`
  ```shell
  journalctl -r
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 28 11:00:00 hostname systemd[1]: Started User Manager for UID 1000.
  Mar 28 10:59:59 hostname systemd[1]: Starting User Manager for UID 1000...
  Mar 28 10:59:58 hostname sshd[5678]: Accepted publickey for user from 192.168.1.100 port 54321 ssh2: RSA SHA256:...
  ...
  ```

- **View a specific number of recent lines**: `journalctl -n 100` (shows last 100 lines)
  ```shell
  journalctl -n 10
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 28 10:59:59 hostname systemd[1]: Starting User Manager for UID 1000...
  Mar 28 11:00:00 hostname systemd[1]: Started User Manager for UID 1000.
  Mar 28 11:00:01 hostname CRON[7890]: (root) CMD (command -v  run-parts >/dev/null && run-parts --report /etc/cron.hourly)
  ...
  ```

## 2. Real-time Monitoring

- **Follow new logs (like `tail -f`)**: `journalctl -f`
  ```shell
  journalctl -f
  ```
  ```output
  # Output will continuously update with new log entries.
  # Press Ctrl+C to exit.
  ```

- **Follow logs for a specific service**: `journalctl -u <service_name> -f`
  ```shell
  # Example: Follow Nginx logs in real-time
  journalctl -u nginx.service -f
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC. --
  Mar 28 11:00:01 hostname nginx[1235]: server started
  Mar 28 11:00:05 hostname nginx[1235]: accepted connection
  ...
  # Output will continuously update with new Nginx log entries.
  # Press Ctrl+C to exit.
  ```

## 3. Filtering by Service or Unit

- **View logs for a specific service**: `journalctl -u <service_name>`
  ```shell
  # Example: View logs for the Apache web server
  journalctl -u apache2.service
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:30:00 hostname systemd[1]: Starting The Apache HTTP Server...
  Mar 27 10:30:01 hostname apachectl[1000]: AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
  Mar 27 10:30:01 hostname systemd[1]: Started The Apache HTTP Server.
  ...
  ```

- **View logs for multiple services**: `journalctl -u <service1> -u <service2>`
  ```shell
  # Example: View logs for Nginx and Apache
  journalctl -u nginx.service -u apache2.service
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:30:00 hostname systemd[1]: Starting A high performance web server and a reverse proxy server...
  Mar 27 10:30:00 hostname systemd[1]: Starting The Apache HTTP Server...
  Mar 27 10:30:01 hostname nginx[1235]: nginx version: nginx/1.18.0 (Ubuntu)
  Mar 27 10:30:01 hostname apachectl[1000]: AH00558: apache2: Could not reliably determine...
  ...
  ```

## 4. Filtering by Time Range

- **View logs since a specific time**: `journalctl --since "YYYY-MM-DD HH:MM:SS"`
  ```shell
  journalctl --since "2026-03-28 10:30:00"
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 28 10:30:00 hostname systemd[1]: Started Snap Daemon.
  Mar 28 10:30:01 hostname systemd[1]: Starting Self-healing of snapd state...
  ...
  ```

- **View logs since "yesterday", "today", or "relative"**:
    - `journalctl --since yesterday`
      ```shell
      journalctl --since yesterday
      ```
      ```output
      # Logs from yesterday until now.
      ```
    - `journalctl --since "1 hour ago"`
      ```shell
      journalctl --since "1 hour ago"
      ```
      ```output
      # Logs from 1 hour ago until now.
      ```
    - `journalctl --since "2 days ago"`
      ```shell
      journalctl --since "2 days ago"
      ```
      ```output
      # Logs from 2 days ago until now.
      ```

- **View logs within a range**: `journalctl --since "YYYY-MM-DD" --until "YYYY-MM-DD HH:MM:SS"`
  ```shell
  journalctl --since "2026-03-27" --until "2026-03-27 15:00:00"
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Fri 2026-03-27 15:00:00 UTC. --
  Mar 27 10:00:00 hostname systemd[1]: Starting ...
  ...
  Mar 27 14:59:59 hostname systemd[1]: Reached target Timer Units.
  ```

## 5. Filtering by Priority/Log Level

- **Show only errors or higher**: `journalctl -p err -b`
  ```shell
  journalctl -p err -b
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:00:00 hostname kernel: DMAR: [Firmware Bug]: No firmware reserve region can be found (bug 134267)
  Mar 27 10:00:00 hostname kernel: tsc: Unable to calibrate against APICIC (buggy BIOS?)
  ...
  ```

- **Log levels**: `0: emerg`, `1: alert`, `2: crit`, `3: err`, `4: warning`, `5: notice`, `6: info`, `7: debug`
- **Example (show warnings and errors)**: `journalctl -p 3..4`
  ```shell
  journalctl -p 3..4
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:00:00 hostname kernel: DMAR: [Firmware Bug]: No firmware reserve region can be found (bug 134267)
  Mar 27 10:00:00 hostname systemd[1]: Mounting Mount user-specific directories...
  Mar 27 10:00:00 hostname systemd[1]: user@1000.service: Failed to set fds for OOMScoreAdjust: Invalid argument
  ...
  ```

## 6. System and Kernel Logs

- **View kernel logs (like `dmesg`)**: `journalctl -k`
  ```shell
  journalctl -k
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:00:00 hostname kernel: Linux version 5.4.0-77-generic (buildd@lcy02-amd64-CI-build)
  Mar 27 10:00:00 hostname kernel: Command line: BOOT_IMAGE=/boot/vmlinuz-5.4.0-77-generic root=UUID=... ro quiet splash vt.handoff=7
  Mar 27 10:00:00 hostname kernel: Kernel command line: BOOT_IMAGE=/boot/vmlinuz-5.4.0-77-generic root=UUID=... ro quiet splash vt.handoff=7
  ...
  ```

- **View logs from the current boot**: `journalctl -b`
  ```shell
  journalctl -b
  ```
  ```output
  -- Logs from the current boot (d4e0b0b0...d4e0b0b0) --
  Mar 27 10:00:00 hostname kernel: Linux version 5.4.0-77-generic (buildd@lcy02-amd64-CI-build)
  Mar 27 10:00:00 hostname systemd[1]: Hostname Service was skipped because of an unmet condition check (ConditionPathIsMountPoint=/etc/hostname).
  ...
  ```

- **List previous boots**: `journalctl --list-boots`
  ```shell
  journalctl --list-boots
  ```
  ```output
  -1 6b8b45a0e9...d4e0b0b0 Mon 2026-03-23 09:00:00 UTC—Mon 2026-03-23 18:00:00 UTC
   0 d4e0b0b0e9...f7c2d1d1 Fri 2026-03-27 10:00:00 UTC—Sat 2026-03-28 11:00:00 UTC
  ```

- **View logs from a previous boot**: `journalctl -b -1` (where -1 is the index from the list)
  ```shell
  journalctl -b -1
  ```
  ```output
  -- Logs from boot 6b8b45a0... (Mon 2026-03-23 09:00:00 UTC) --
  Mar 23 09:00:00 hostname kernel: Booting Linux on physical CPU 0x0
  Mar 23 09:00:00 hostname kernel: Linux version 5.4.0-77-generic (buildd@lcy02-amd64-CI-build)
  ...
  ```

## 7. JSON and Other Output Formats

- **Output in JSON format**: `journalctl -o json`
  ```shell
  journalctl -n 1 -o json
  ```
  ```output
  {
      "__CURSOR" : "s=...;i=...;b=...;m=...;t=...;x=...",
      "__REALTIME_TIMESTAMP" : "1678886400123456",
      "__MONOTONIC_TIMESTAMP" : "123456789",
      "_BOOT_ID" : "d4e0b0b0e9...f7c2d1d1",
      "_MACHINE_ID" : "abcde123...abcde123",
      "_HOSTNAME" : "hostname",
      "_TRANSPORT" : "syslog",
      "MESSAGE" : "CRON[7890]: (root) CMD (command -v  run-parts >/dev/null && run-parts --report /etc/cron.hourly)",
      "_PID" : "7890",
      "_UID" : "0",
      "_GID" : "0",
      "_COMM" : "CRON",
      "_EXE" : "/usr/sbin/cron",
      "_CMDLINE" : "/usr/sbin/cron -f",
      "_CAP_EFFECTIVE" : "0",
      "_SYSTEMD_CGROUP" : "/system.slice/cron.service",
      "_SYSTEMD_UNIT" : "cron.service",
      "_SYSTEMD_SLICE" : "system.slice",
      "_SELINUX_CONTEXT" : "unconfined_u:system_r:cron_t:s0-s0:c0.c1023",
      "SYSLOG_IDENTIFIER" : "CRON"
  }
  ```

- **Output in a more readable JSON format**: `journalctl -o json-pretty`
  ```shell
  journalctl -n 1 -o json-pretty
  ```
  ```output
  # Similar to json output, but nicely indented and human-readable.
  ```

- **Output in short format (default)**: `journalctl -o short`
  ```shell
  journalctl -n 1 -o short
  ```
  ```output
  Mar 28 11:00:01 hostname CRON[7890]: (root) CMD (command -v  run-parts >/dev/null && run-parts --report /etc/cron.hourly)
  ```

## 8. Journal Maintenance

- **Check disk usage of logs**: `journalctl --disk-usage`
  ```shell
  journalctl --disk-usage
  ```
  ```output
  Archived and active journals take up 1.5G in the file system.
  ```

- **Clean up logs (keep only last 1GB)**: `journalctl --vacuum-size=1G`
  ```shell
  sudo journalctl --vacuum-size=1G
  ```
  ```output
  Vacuuming done, freed 500.0M of journal files.
  ```

- **Clean up logs (keep only last 7 days)**: `journalctl --vacuum-time=7d`
  ```shell
  sudo journalctl --vacuum-time=7d
  ```
  ```output
  Vacuuming done, freed 700.0M of journal files.
  ```
