---
layout: post
title: "git study summary"
date: 2012-05-12 00:00
comments: true
categories: [git]
tags: [git]
---

<!-- more -->
##create repo

```
1087 git init —bare study.git <=== crate study.git 
1088 ls study.git/ 1089 cat study.git/config 
1090 ls 1091 git clone study.git/ 
1092 ls 
1093 cd study 
1094 ls 
1095 touch study.readme 
1096 echo “This is readme for study(master)” 
1097 echo “This is readme for study(master)” >>study.readme 
1098 git add study.readme 
1099 git commit -s study.readme <===== commit a file for test. 
1100 git push origin master <===== push them to the server(locally). 
1101 cd .. 1102 ls
```

##create a remote branch.

```
1103 git clone study.git/ tmp1 
1104 cd tmp1/ 
1105 ls 
1106 git log 
1107 cd .. 
1108 ls 
1109 cd study 
1110 ls 
1111 git checkout -b dev_junwei <======== create a locally branch. 
1112 ls 
1113 vim study.readme 
1114 ls 
1115 git commit study.readme 
1116 git mv study.readme study.dev.junwei.readme 
1117 git push origin dev_junwei <======== create a remote branch. and remote branch has same name with the local one.
1118 git branch
```

##create a mirror for the study.git

```
git clone —bare study.git/ mirror.git
```

##Add a new remote git resp.

 ```
 1134  git remote add mirror /home/junwei/git_study/mirror.git/  <=== add a new remote and name it as mirror.
 1136  git remote show
 1137  git remote show origin 
 1138  git remote show mirror 
 1139  git status
 1140  git fetch mirror  <===== !!!! important, Only thus following checkout could be sucess

 1141  git checkout  -b m_master mirror/master  <=== create local branch according remote branch.
 1142  git checkout  -b m_dev mirror/dev_junwei 

 1164  git commit -sa <=== commit a change to local branch.

 1168  git push mirror  m_dev:dev_junwei <=== push the local change to remote branch. "m_dev" is created in 1142.

 1174  git push mirror m_dev:dev_junwei  <=== do some change/commit and push again to remote mirror.
             FU CK( NOT m_dev/dev_junwei)!!!!!!!

 1176  git status
```

git config push.default tracking 来让git push命令默认push当前的分支到对应的remote tracking分支上

```
[junwei@junwei study]$ git config push.default tracking
[junwei@junwei study]$ cat .git/config 
....
[push]
 default = tracking
[junwei@junwei study]$ 

git push origin :remote_branch_name
```
```
[martin@fc16 example]$ git log --oneline  
7f9df38 This file should only be seen on branch 'dev'.
5679bd2 fill a line in readme file on branch 'dev'.
3d83de1 Add an empty readme
[martin@fc16 example]$ 
[martin@fc16 example]$ git tag init 3d83de1
[martin@fc16 example]$ git tag v1.0 
[martin@fc16 example]$  
[martin@fc16 example]$ git tag
init
v1.0
[martin@fc16 example]$ git push
Everything up-to-date
[martin@fc16 example]$ git push --tags
Total 0 (delta 0), reused 0 (delta 0)
To /var/lib/git/test/example.git/
 * [new tag]         init -> init
 * [new tag]         v1.0 -> v1.0
[martin@fc16 example]$ 
```
##delete remote tags.
```
[martin@fc16 example]$ git push  origin  :v1.a0
To /var/lib/git/test/example.git/
 - [deleted]         v1.a0
[martin@fc16 example]$ 
```
