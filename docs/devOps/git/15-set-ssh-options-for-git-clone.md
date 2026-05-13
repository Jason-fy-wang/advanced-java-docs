---
tags:
  - git
  - git-ssh
  - ssh-options
---

Set ssh options for git clone and also add timeout .
```
GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" timeout 10m git clone git@github.com:USERNAME/REPOSITORY.git
``` 