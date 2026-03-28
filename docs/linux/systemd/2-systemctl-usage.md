---
tags:
  - systemctl
  - systemd
---

# Summary of Common systemctl Commands

`systemctl` is the primary tool for managing systemd systems and services. Below is a summary of commonly used commands in daily work.

## 1. Service Lifecycle Management

- **Start a service**: `systemctl start <service>`
  ```shell
  # Example: Start the Nginx web server
  sudo systemctl start nginx
  ```
  ```output
  # No output on success
  ```

- **Stop a service**: `systemctl stop <service>`
  ```shell
  # Example: Stop the Nginx web server
  sudo systemctl stop nginx
  ```
  ```output
  # No output on success
  ```

- **Restart a service**: `systemctl restart <service>`
  ```shell
  # Example: Restart the Nginx web server
  sudo systemctl restart nginx
  ```
  ```output
  # No output on success
  ```

- **Reload configuration**: `systemctl reload <service>` (reloads configuration files without stopping the service)
  ```shell
  # Example: Reload Nginx configuration
  sudo systemctl reload nginx
  ```
  ```output
  # No output on success
  ```

- **Check service status**: `systemctl status <service>`
  ```shell
  # Example: Check the status of Nginx
  systemctl status nginx
  ```
  ```output
  ● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-03-27 10:30:00 UTC; 1 day ago
       Docs: man:nginx(8)
    Process: 1234 ExecStart=/usr/sbin/nginx -g "daemon on; master_process on;" (code=exited, status=0/SUCCESS)
   Main PID: 1235 (nginx)
      Tasks: 3 (limit: 4686)
     Memory: 6.5M
     CGroup: /system.slice/nginx.service
             ├─1235 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             ├─1236 nginx: worker process
             └─1237 nginx: worker process
  ...
  ```

## 2. Boot-up Configuration Management

- **Enable service on boot**: `systemctl enable <service>`
  ```shell
  # Example: Enable Nginx to start on boot
  sudo systemctl enable nginx
  ```
  ```output
  Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /lib/systemd/system/nginx.service.
  ```

- **Disable service on boot**: `systemctl disable <service>`
  ```shell
  # Example: Disable Nginx from starting on boot
  sudo systemctl disable nginx
  ```
  ```output
  Removed /etc/systemd/system/multi-user.target.wants/nginx.service.
  ```

- **Disable and stop immediately**: `systemctl disable --now <service>`
  ```shell
  # Example: Disable and stop Nginx
  sudo systemctl disable --now nginx
  ```
  ```output
  Removed /etc/systemd/system/multi-user.target.wants/nginx.service.
  ```

- **Check if service is enabled**: `systemctl is-enabled <service>`
  ```shell
  # Example: Check if Nginx is enabled
  systemctl is-enabled nginx
  ```
  ```output
  enabled
  # or
  disabled
  ```

## 3. Status Checking and Unit Listing

- **List all running services**: `systemctl list-units --type=service`
  ```shell
  systemctl list-units --type=service
  ```
  ```output
  UNIT                          LOAD   ACTIVE SUB     DESCRIPTION
  apache2.service               loaded active running The Apache HTTP Server
  containerd.service            loaded active running containerd container runtime
  cron.service                  loaded active running Regular background program processing daemon
  dbus.service                  loaded active running D-Bus System Message Bus
  ...
  ```

- **List all services (including inactive ones)**: `systemctl list-units --type=service --all`
  ```shell
  systemctl list-units --type=service --all
  ```
  ```output
  UNIT                           LOAD   ACTIVE   SUB     DESCRIPTION
  alsa-restore.service           loaded inactive dead    Restore Sound Card State
  alsa-state.service             loaded active   running Manage Sound Card State (restore and save)
  apache2.service                loaded active   running The Apache HTTP Server
  ...
  ```

- **List all installed unit files**: `systemctl list-unit-files --type=service`
  ```shell
  systemctl list-unit-files --type=service
  ```
  ```output
  UNIT FILE                                  STATE           VENDOR PRESET
  auditd.service                             enabled         enabled
  autovt@.service                            disabled        enabled
  blk-availability.service                   disabled        disabled
  ...
  ```

- **Check if a service is active**: `systemctl is-active <service>`
  ```shell
  systemctl is-active nginx
  ```
  ```output
  active
  # or
  inactive
  ```

- **Check for failed services**: `systemctl --failed`
  ```shell
  systemctl --failed
  ```
  ```output
    UNIT          LOAD   ACTIVE SUB    DESCRIPTION
  ● failed.service loaded failed failed Failed Service Example

  LOAD   = Reflects whether the unit definition was properly loaded.
  ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
  SUB    = The low-level unit activation state, values depend on unit type.
  1 loaded units listed.
  ```

## 4. Unit Files and Configuration Editing

- **Edit service unit**: `systemctl edit <service>` (creates/edits a drop-in override snippet)
  ```shell
  # Example: Edit Nginx service
  sudo systemctl edit nginx
  ```
  ```output
  # Opens an editor (e.g., nano or vi) with a file like:
  # /etc/systemd/system/nginx.service.d/override.conf
  # You can add overrides here, e.g.:
  # [Service]
  # ExecStartPost=/bin/sleep 1
  ```

- **Edit full unit file**: `systemctl edit --full <service>`
  ```shell
  # Example: Edit the full Nginx service file
  sudo systemctl edit --full nginx
  ```
  ```output
  # Opens an editor with the main service file:
  # /lib/systemd/system/nginx.service
  # Make changes cautiously here.
  ```

- **Specify editor for editing**: `SYSTEMD_EDITOR=vim systemctl edit <service>`
  ```shell
  # Example: Use vim to edit the Nginx service
  sudo SYSTEMD_EDITOR=vim systemctl edit nginx
  ```
  ```output
  # Opens vim editor for the drop-in override.
  ```

- **Reload systemd manager**: `systemctl daemon-reload` (required after manual changes to `.service` files)
  ```shell
  sudo systemctl daemon-reload
  ```
  ```output
  # No output on success
  ```

<h2> 5. Masking and Locking </h2>

- **Mask a service**: `systemctl mask <service>` (links to `/dev/null`, preventing it from being started manually or automatically)
  ```shell
  # Example: Mask the Nginx service
  sudo systemctl mask nginx
  ```
  ```output
  Created symlink /etc/systemd/system/nginx.service → /dev/null.
  ```

- **Unmask a service**: `systemctl unmask <service>`
  ```shell
  # Example: Unmask the Nginx service
  sudo systemctl unmask nginx
  ```
  ```output
  Removed /etc/systemd/system/nginx.service.
  ```

<h2> 6. Log Viewing (with journalctl) </h2>

- **View logs for a specific service**: `journalctl -u <service>`
  ```shell
  # Example: View Nginx logs
  journalctl -u nginx.service
  ```
  ```output
  -- Logs begin at Fri 2026-03-27 10:00:00 UTC, end at Sat 2026-03-28 11:00:00 UTC. --
  Mar 27 10:30:00 hostname systemd[1]: Starting A high performance web server and a reverse proxy server...
  Mar 27 10:30:01 hostname nginx[1235]: nginx version: nginx/1.18.0 (Ubuntu)
  Mar 27 10:30:01 hostname nginx[1235]: built with OpenSSL 1.1.1f  31 Mar 2020
  Mar 27 10:30:01 hostname systemd[1]: Started A high performance web server and a reverse proxy server.
  ...
  ```

- **Real-time scrolling view**: `journalctl -u <service> -f`
  ```shell
  # Example: Follow Nginx logs in real-time
  journalctl -u nginx.service -f
  ```
  ```output
  # Output will continuously update with new Nginx log entries.
  # Press Ctrl+C to exit.
  ```
