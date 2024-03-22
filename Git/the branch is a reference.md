## Preface

In git, the branch is a pointer to a particular snapshot. We can think of it as an indicator of a top of a cluster of commits.

## Branch is a type of a reference

A branch is a pointer to a commit, every time we make a commit, the branch pointer moves forward.when using the --points-at argument,we can get a list of branches that point to a specific commit.

```shell
git branch --points-at commit-hash
```

In a local repository, the .git/refs/heads directory contains all of our local branches. we can use the command to list all the branches.

```shell
ls ./.git/refs/heads/
```

## About HEAD

when executing the git log command, we can see a list of branches

```shell
commit 5b5bb249e4990e672a96bbe4800a6e36d9a60962 (HEAD -> master, origin/master, origin/HEAD)
```

Simplifying it, HEAD is a pointer to a commit that our repository is checked out on.it most cases means that HEAD points to the same commit that a branch that we currently use.If we make a commit,HEAD now points to it.

But there exists some exception, named as detached head. When using git checkout, we can provide a hash of a specific commit.

```shell
git checkout 3238d85
```

this command will puts us in a detached HEAD state. It means that HEAD will no longer points to a specific branch.

## Git reset

Understanding the HEAD can come in handy.An example is deleting an unpushed commit. We can use reset command like this:
```shell
git reset --soft HEAD~1

git reset --hard HEAD~1
```
this command will reset the HEAD to a specified state.By providing HEAD~1, we point to a parent of the last commit,by doing so,we remove the last commit that we have made.

we could also delete more than just one commit,for example,by typing HEAD~4 and removing four commits.

We can check what the HEAD points to by looking up the HEAD file in the .git directory
```shell
cat ./.git/HEAD
```

## Packed-refs

As the repository grows significantly,out of performance,git periodically compresses refs into a single file.By doing so, git moves all branches and tags into a single packed-refs file.When this happens, the directory .git/refs will looks empty,but you can see all the information in packed-refs file.
```shell
cat ./.git/packed-refs
```

