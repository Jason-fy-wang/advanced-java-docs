---
tags:
  - podman
  - error
  - podman-issue
---
podman bring up container get error:
```shell
podman run -d --name mockservice -p 8080:8080 21d4518e1853
Error: OCI runtime error: crun: error `creating` systemd unit `libpod-5b0c1fa96eac81d596a05f86648f84a1f7ad782423d5667ce2fd8da04e927ba5.scope`: got `failed`
```

```shell
# check help from podman, we can see podman cgroup-manager default is system. 
Options:
--cgroup-manager string       Cgroup manager to use ("cgroupfs"|"systemd") (default "systemd")
```

```shell
## check system
systemctl --user status

### show fails
systemctl --user --fail

### reset fails
systemctl --user reset-fails


## switch to cgroups
podman run -d --name mockservice -p 8080:8080 --cgroups=disabled  21d4518e1853
```


try to find the error with journalctl
```shell
## get the user error message,can give some clue about issue
journalctl --user -xe --no-pager
```

get error below:
```text
systemd[2103]: podman-2581895.scope: Couldn't move process 2581895 to requested cgroup '/user.slice/user-1000.slice/user@1000.service/user.slice/podman-
2581895.scope' (directly or via the system bus): Input/output error
Aug 20 10:32:12 global systemd[2103]: podman-2581895.scope: Failed to add PIDs to scope's control group: Permission denied
```

`Permission denied`: the root cause here.


reset your user level systemd:
```shell
systemctl --user daemon-reexec
systemctl --user reset-failed

# after reset done, below should run without error
systemd-run --user --scope --same-dir /bin/true
```

