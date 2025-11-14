---
tags:
  - git
  - commit-history
---
For some company internal requirement,  the commit history for one change should be longer than 3 days.  So what should we do if the change last logner that 3days?

Yes, we have to shorten the commit history.

There are two day to achieve this.

### merge commit history through rebase

This method is simple, before we merge to master branch, we can leverage `git rebase` to merge the commit history.  Then `git push origin HEAD --force` to refresh the feature commit history.





### re-create feature branch

This one is more easier,  delete the feature branch (remote and local) and  push  the change to feature branch again.


