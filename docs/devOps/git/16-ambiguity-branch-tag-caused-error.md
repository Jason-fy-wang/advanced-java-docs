---
tags:
  - git
  - branch
  - tag
  - ambiguity-tag
---
In our new project, we create branch base on every new feature, and the branch name is the release tag.  Which means, the tag name is same as the branch name.

For release, we will use the last release version as the rollback version, all the method is make scense.

But recently, we get the rollback version has prefix `tags/`,  this prefix will cause error during the automate job.

So we try to analyze the root cause. 
```shell
# this is the git operation we use to get the last release version
git for-each-ref --sort=-creatordate --count=5 --format='%(refname:short)'  refs/tags/v*
v3
v2
v1

# the above is normal output, we can get the release tags.
```

The above is the normal output, but sometimes we get the value: 
```shell
tags/v3
tags/v2
v1
```

After double checked, the tag name is correct, they are named as expect. (v3,v2).

But the result display incorrect, why ?
The reason is git will try to output the result `disambiguity`. so to achieve this, git keeps the `tags/` prefix to make it disambiguity.  And the corresponding branch `v3, v2` not delete after it merge to main branch. 
So the final reason is there have a branch with same name as tag, git add the prefix `tags/` to make the output disambiguity.

So we update the format as `--format='%(refname:lstripe=2)'`


