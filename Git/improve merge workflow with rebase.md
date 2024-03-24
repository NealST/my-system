## The Problem with using merge

When using git merge to merge the changes from other branches, if current branch has some different commits, this merge action will create a new merged commit in history. It will do harm to the readability of the git log. If we want to avoid it, rebase is a good alternative.

## Rebase can improve it

Take an assumption that there are two branches named as master and feature-2, master has some commits that do not exist in feature-2, at the same time, feature-2 also has some new commits.If we want to merge feature-2 into master and we want to avoid the problem of git merge. we could use git rebase like this:

```shell
git checkout feature-2

git rebase master

git checkout master

git merge feature-2
```

These four steps will help us to achive the goal. The reason is that when executing rebase command in feature-2 branch, it will rewrite the commit history of feature-2 and turn the history into a straightforward line based on the latest commit of master,so when checkouting back master branch and executing merge command, it's not necessary for master to create a new merged commit to merge the different commit history.it can use the latest commit in feature-2 as the HEAD directly.

After executing rebase command, we need to push the rebased changes to the remote branch. If we execute the command
```shell
git push origin feature-2
```
we will encounter failure. The reason for this failure is that the history of out local feature-2 branch has been rewrote,so it no longer matches its remote counterpart.To solve this problem,the most straightforward way is to force push it.
```shell
git push --force origin feature-2
```
But this force option is not safe, if someone else is pushing their changes to feature2 branch, --force flag would cause all of that work to be erased.Instead, we should use the --force-with-lease flag.
```shell
git push --force-with-lease origin feature-2
```
This flag is safer, because it refuses to update the branch if somebody updated the remote branch.

## When not to use rebase

Because of the reason that rebase action will rewrite the commit history, rebase is not allowed to be used in some public branches, it may results to the failure when your teammates pushing his changes to the public branch due to the change of commit history, Git will think that the history of the remote public branch diverged from the local copies that your teammates maintain.
