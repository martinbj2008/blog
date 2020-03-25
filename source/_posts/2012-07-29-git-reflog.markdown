---
layout: post
title: "通过git reflog 找回的commit"
date: 2012-07-29 00:00
comments: true
categories: [git]
tegs: [git, reflog]
---

<!-- more -->
## Prepare
```
[martin@fc17 git_study]$ git log --oneline 
83e7d89 add lineb in readme <== this commit will be reset hardly
4fd6367 add line1 in readme
bfd590e touch readme
```

```
[martin@fc17 git_study]$ git reset HEAD^ --hard 
HEAD is now at 4fd6367 add line1 in readme
[martin@fc17 git_study]$ git log --oneline 
4fd6367 add line1 in readme
bfd590e touch readme
```

```
[martin@fc17 git_study]$  cat readme 
line a
通过git reflog 我们可以得到误删除的commit对应的ID
```

### git reflog

```
[martin@fc17 git_study]$ git reflog 
4fd6367 HEAD@{0}: reset: moving to HEAD^
83e7d89 HEAD@{1}: commit: add lineb in readme
4fd6367 HEAD@{2}: commit: add line1 in readme
bfd590e HEAD@{3}: commit (initial): touch readme
```

有了commit ID, 一切就OK了。想干啥都行了 cherry-pick, checkout …

```
 [martin@fc17 git_study]$ git checkout -b try_recover 83e7d89 
 Switched to a new branch ‘try_recover’ 
 [martin@fc17 git_study]$ git log —oneline
  83e7d89 add lineb in readme 
  4fd6367 add line1 in readme 
  bfd590e touch readm
 [martin@fc17 git_study]$ cat readme 
 line a 
 line b
```


