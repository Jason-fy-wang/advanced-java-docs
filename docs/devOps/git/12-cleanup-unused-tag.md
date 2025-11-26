---
tags:
  - cleanup
  - clean-tag
---


```shell
# clean local tag
git tag | grep 'tag_name' && git tag -d tag_name 
# clean remote tag
git ls-remote -t | grep 'tag_name' && git push --delete tag_name 

```

