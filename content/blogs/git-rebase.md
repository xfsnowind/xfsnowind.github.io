---
title: "Learning Notes - Git rebase vs Git merge"
date: 2022-11-01T20:47:47+08:00
author: "Feng Xue"
draft: false
tags: ["Learning Notes", "Git"]
toc: true
---

## Git merge

Normally, to get the latest update from `main` branch during development the feature or fix branch, I would checkout to the `main` branch and `git pull` the latest commits and then checkout back and run the merge command.

```bash
git checkout main
git pull
git checkout FEATURE-BUG-BRANCH
git merge --no-ff development

or 

git checkout main
git pull
git merge FEATURE-BUG-BRANCH main
```

It works and will create a `MERGE` commit in the feature branch. It's OK because it's *non-destructive* operation. But it will always have an extraneous merge commit in the history, which may be fine to have it at the end of development, but not good during the development. So is there a possibility to merge the `main` branch into our feature branch without a merge commit?

## Git rebase

The answer is `git rebase`. We can try the following:

```
git checkout FEATURE-BUG-BRANCH
git rebase main
```

This will move the whole feature branch to beginning of the `main` branch. And instead of creating a merge commit, it will *re-write* the whole history by making new commits, even there is some merge commits in the feature branch before.

The benefit of rebasing would be, first, there is no unrequired merge commits, second, the git history is quite linear, the main branch would be behind the feature branch. However, all of these should be done when there is only one developer, no collaborator. Because rebasing would *re-write* the commits, so if you collaborate with other developers, the commits from main branch would be different from the public `main` branch. That would be a hard situation.

So before using `git rebase`, ask yourself, is there another developer working together with you on this branch. If the answer is yes, then use `merge` instead.


## What happen when rebase mixed with merge
Let's do some experiment, creating a `main` and `feature` branch.

```bash
> git checkout -b main
Switched to a new branch 'main'
(main)> touch test.js
(main)> git add test.js
(main)> git commit -m "init"
[main (root-commit) b7c27e8] init
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test.js

// feature branch
(main)> git checkout -b feature
Switched to a new branch 'feature'
(feature)> touch feature.js
(feature)> git add feature.js
(feature)> git commit -m "feature.js"
[feature 4d60997] feature.js
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 feature.js
```

So we have one commit in both feature and main branch. Let's create one more commit for each branch.

```bash
(feature)> git co main
Switched to branch 'main'
> touch main.js
(main)> git add main.js
(main)> git commit -m "main.js"
[main c593f25] main.js
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 main.js

(main)> git co feature
Switched to branch 'feature'
(feature)> touch feature2.js
(feature)> git add feature2.js
(feature)> git commit -m "feature2.js"
[feature 193b518] feature2.js
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 feature2.js
```

Now, let's merge the `main` to the `feature` and check how the history looks like

```bash
(feature)> git merge --no-ff main
Merge made by the 'ort' strategy.
 main.js | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 main.js

(feature)> git log --oneline --graph
*   e99a53e (HEAD -> feature) Merge branch 'main' into feature
|\
| * c593f25 (main) main.js
* | 193b518 feature2.js
* | 4d60997 feature.js
|/
* b7c27e8 init
```

Yes, a `merge` commit is created with merging. Let's check how it looks like with rebasing

```bash
(feature)> git co main
Switched to branch 'main'
(main)> touch main2.js
(main)> git add main2.js
(main)> git commit -m "main2.js"
[main 6c43d0f] main2.js
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 main2.js

(main)> git co feature
Switched to branch 'feature'
(feature)> touch feature3.js
(feature)> git add feature3.js
(feature)> git commit -m "feature3.js"
[feature dc5fbcb] feature3.js
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 feature3.js

 (feature)> git rebase main
Successfully rebased and updated refs/heads/feature.
(feature)> git log --oneline --graph
* 3caa16c (HEAD -> feature) feature3.js
* cd53eed feature2.js
* 595209c feature.js
* 6c43d0f (main) main2.js
* c593f25 main.js
* b7c27e8 init
```

We can see the `feature` branch locates on the top of `main` branch and the history is linear, even we actually have created a `merge` commit before. 

And all the commits in the `main` branch are kept, while all the commits in `feature` are, as we said, *re-write* as new commits, we can see the commit ids/hashes are different.

Let's do one more step, merge the `main` to `feature` again.

```bash
(feature)> git co main
Switched to branch 'main'
(main)> touch main-after-rebase.js
(main)> git add main-after-rebase.js
(main)> git commit -m "main after rebase"
[main 92bd62e] main after rebase
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 main-after-rebase.js

 (main)> git co feature
Switched to branch 'feature'
(feature)> touch feature-after-main-after-rebase.js
(feature)> git add feature-after-main-after-rebase.js
(feature)> git commit -m "feature after main after rebase"
[feature 2556a80] feature after main after rebase
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 feature-after-main-after-rebase.js

(feature)> git merge --no-ff main
Merge made by the 'ort' strategy.
 main-after-rebase.js | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 main-after-rebase.js
(feature)> git log --oneline --graph
*   f162af2 (HEAD -> feature) Merge branch 'main' into feature
|\
| * 92bd62e (main) main after rebase
* | 2556a80 feature after main after rebase
* | 3caa16c feature3.js
* | cd53eed feature2.js
* | 595209c feature.js
|/
* 6c43d0f main2.js
* c593f25 main.js
* b7c27e8 init
```

So the `merge` commit is created as expected, but it's based on the `main2.js` commit, not from the beginning.