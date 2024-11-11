---
tags:
  - python
  - pip
---

```shell
## pip-preview
pip install pip-review
### upgrade all dependency
pip-review --auto


## pip-tools
pip install pip-tools

### upgrade requirement.txt dependency
pip-compile --upgrade

### install upgraded denendency
pip install -r requirement.txt


## upgrade with pip
pip freeze > requirements.txt
pip install -r requirements.txt --upgrade


```

