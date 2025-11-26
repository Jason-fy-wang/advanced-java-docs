---
tags:
  - git-squash
  - squash
  - git-commit
---
The normal procedure for develop as below:
	1. checkout feature branch from master
	2. update code change to branch branch
	3. test
	4. fix bug
	5. commit to feature branch
	6. repeat 3 - 5
	7. after feature stable, merge to master.

I believe, every one go throguh above step tons of times.  Its normal, nothing special.

What i want to clarify here is `step 7`.  The best practise, we should squash all change into one commit, then merge it to master.  All this will make master as a `straight line`, and good to review the change.

We can squash the feature branch change into one commit with below script:
```shell
# squash all change
git reset --soft $(git log feature_branch master)
git add .
git commit -m"feature branch change summary"
git push origin feature_branch --force
```

Of course, we can squash the commit history with above script.  But there some exception or limitation need to clarity: 
`Limitation:`
>  situation 1: 
>  Phase 1: the feature branch merged to master branch
>  Phase 2: we need update same line of the file, and run the above script again. 
>   Then we will encounter conflict.

how to resolve this: 
> if we merged before,  we just commit the new change to feature branch only. Wont do any commit history rewrite.

```shell
git log master --oneline --merges | grep feature_branch
merged_exists=$?
if [[ $merged_exists -ne 0 ]]; then
	git reset --soft $(git log feature_branch master)
	git add .
	git commit -m"feature branch change summary"
	git push origin feature_branch --force
else
	git add .
	git commit -m"feature branch change summary"
	git push origin feature_branch
fi
```



