---
layout: post
title: "How to submit kernel patch by git-mail"
date: 2013-06-26 00:00
comments: true
categories: [git]
tags: [git]
---

```
git send-email --annotate  --subject-prefix="PATCH v2 net-next" \
  --in-reply-to=The_previous_email_message_id_to_be_replied \
  --smtp-debug --to MAINTENACE_person_email --cc CCLIST HEAD^^
```

<!-- more -->

1. `—annotate`: 临时增加一些注释到邮件里。添加方法跟git commit --amend 类似, 但是这些注释不会作为标准patch的一部分（即不会被包含到将来的commit message里）。
```
Adding Extra Notes to Patch Mails

Sometimes it's convenient to annotate patches with some notes that are not meant
to be included in the commit message. For example, one might want to write 
"I'm not sure if this should be committed yet, because..." in a patch, but the
text doesn't make sense in the commit message. Such messages can be written
below the three dashes "---" that are in every patch after the commit message.
Use the --annotate option with git send-email to be able to edit the mails 
before they are sent.
```
这些注释要添加在---的后面。
例如:
这样的注释不会被保留到最终的git 仓库里。所以我把它们成为临时注释。
```
37 ---
<=== 使用annotate，在这里添加临时注释。
38  arch/x86/kernel/cpu/mtrr/cleanup.c | 8 ++++----
39  1 file changed, 4 insertions(+), 4 deletions(-)
```

2. Cc: .....
3. Signed-off-by: ....

4. `—subject-prefix`: 邮件标题前面中括号里的内容。 要是邮件不小心发错了，需要重发时。要带有”RESEND”字样。 “PATCH vX YYYY”, X标识版本号。第一个版本时可以省略x及前面的v. YYYY 标识具体的那个模块。对网络模块来说，重要的问题发送net， 不重要的发送到net-next。

5. `—in-reply-to` 主要用在邮件讨论上。 使邮件保持在一个thread里。
```
Message-ID to be used as In-Reply-To for the first email?
This can usually be left empty, in which case the patch or patches will appear
as a new thread. When sending updated patches, in my opinion it's nice to set
this to the "Message-Id" header of the previous version of the patch
(for multi-patch series, use the cover letter's Message-Id), so that the full
patch history can be easily accessed. This is the only valid use case for
setting the "In-Reply-To" header (again, in my opinion). In other words, please
do NOT send patches as replies to regular discussion.
```
6. `HEAD^^`: means two patch to be sent. 当发送一系列的patch时有用。

关于维护人。 有两种方式可以得到， kerel源文件根目录下的MAINTAINERS 或者./scripts/get_maintainer.pl

```
$ cat ~/.gitconfig
[user]
  name = XXXX
  email = martinbj2008@xxxx.com
[sendemail]
  smtpencryption = tls
  smtpserver = smtp.gmail.com
  smtpuser = martinbj2008@xxxx.com
  smtpserverport = 587
  confirm = always
  smtppass=XXXXXXXXXXXX
  from=martinbj2008@xxxx.com
```
