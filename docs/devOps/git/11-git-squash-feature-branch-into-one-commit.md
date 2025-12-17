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
`Limitation 1:`
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


`Limitation 2:`
> 1. we create feature branch-a and update our code
> 2. others create feature branch-b and update code , finally merge to master before branch-a
> 3. if we still use `git reset --soft` to rewirte the history, our Pull request will contain branch-b change as well.
> 
> So  what we can do to include our change only? 


here what we do:

```shell
Soluation 1:
# sync from master
git pull origin master
# reset to original branch
git reset --soft $(git log feature_branch master)
# reset local stach space
git reset .
# add what we changed only, f.g. we changed fileA and fileB
git add *fileA *fileB
# reset other changes, include our change only
git checkout .
git commit -m"feature branch change summary"
git push origin feature_branch --force
```


```shell
Solution 2:

# create new branch
git checkout master
git checkout -c branch_squash

git merge --squash branch_a

git commit -m"change all"
git push origin branch_squash

# then create pull request for branch_squash.
```




