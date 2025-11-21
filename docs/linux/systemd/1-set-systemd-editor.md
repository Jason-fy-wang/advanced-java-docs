---
tags:
  - systemd
  - systemd-editor
---

```shell
# edit system-service, default edit with nano.
sudo systemctl edit serviceName
sudo SYSTEMD_EDITOR=vim systemctl edit ollama

# set environment variable:
SYSTEMD_EDITOR=vim
EDITOR=vim
VISUAL=vim

# set variable in /etc/environment to set global environement

```




