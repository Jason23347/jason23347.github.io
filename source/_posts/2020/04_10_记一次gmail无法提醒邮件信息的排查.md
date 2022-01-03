---
title:  "记一次GMAIL无法提醒邮件信息的排查"
date:   2020-04-10 19:59:23
tags:
  - shell
  - fetchmail
  - cron
categories:
  - 折腾
---

> 最近发现手机上GMAIL总是不提醒新的邮件，并且消息都变成了已读。
这使得我总是收不到来自github和gitlab的消息通知。
这个问题很早就出现过了，最近出现得十分频繁，今天我把它解决了。

#### 邮箱服务器与客户端

首先我思考了以下我最近关于邮件做了哪些设置，邮件服务器是我自己配的，客户端则有三个：

- Ubuntu上的`mutt`+`fetchmail`+`maildrop`
- Andriod上的`GMAIL`
- Microsoft Flow

其中最后一个是手机上一个提供简易自动化任务的软件，一个月前我设置过检测GMAIL邮件并弹出提示，不过因为不好用就把任务删除了。
再次检查确定没有任务读取GMAIL，虽然它看起来最可疑，但却是最先被排除的。
其次思考GMAIL，GMAIL可能会收到消息但不提醒，但是不会收到消息直接变成已读，所以先保留。
最后就是fetchmail的检查了。

#### cron任务与fetchmail

以man page的自我介绍来说，`fetchmail`  is  a mail-retrieval and forwarding utility.

我用它来拉取邮件列表，cron任务如下所示
```bash
*/5 * * * * fetchmail
```

然后我恍然大悟，原本的cron任务是每15分钟一次，现在被我改成了5分钟一次，自此我的GMAIL才无法正常获取邮件列表。

#### 真相大白

原来问题处在fetchmail身上！
GMAIL能够设置的最短拉取邮件列表的时间就是15分钟，以往两者的时间间隔相等，所以GMAIL有一定的的概率先拉取邮件，但是当fetchmail查询间隔变为5分钟之后，GMAIL就没有一丝机会获取到新的邮件了。

#### 问题解决

刚才的讨论推测出了问题的起因，但是没有涉及到问题的本质，那就是，为什么fetchmail拉取邮件之后，邮件被标记成已读？
通过查阅资料得知，

- fetchmail标记信息为已读的问题自古就已经存在
- 有的人没有办法，有的人自己写了插件

我怎么办呢，我端详我的fetchmail配置文件，在里面找到一个关键字，`imap`。
编辑`~/.fetchmailrc`：

```conf
defaults
mda "/usr/bin/maildrop"

poll imap.jason233.top
proto imap
port 143
user jason@jason233.top
password ******
mimedecode
keep
```

众所周知，IMAP和POP3都是邮件服务器常用的协议，对于两者的详细区别不再赘述。
通俗来讲，POP3是“只读”的，IMAP是“同步修改”的。
所以，把IMAP换成POP3就可以了。

只要过了这一波测试...还不行。
不过小场面，不要慌，继续查阅资料，添加一个UIDL参数就可以了。

```bash
defaults
mda "/usr/bin/maildrop"

poll pop3.jason233.top
proto po3
port 110
uidl
user jason@jason233.top
password ******
mimedecode
keep
```

#### 进阶crontab

通过cron任务发桌面提醒我有新邮件。

cron:
```bash
*/5 * * * * /home/jason/.local/bin/fetchmail-and-notify
```

编辑文件`fetchmail-and-notify`：

```bash
#!/bin/bash

# env for notify-send
export DISPLAY=:0
export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"

# config
MAIL_INBOX=~/mail/personal/jason/INBOX
LOG_FILE=~/mail/log/notify.log
TIMESTAMP_FILE=~/mail/last_check

# open log stream (append)
exec 3>>$LOG_FILE
write_log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" >&3
}

# fetch mail
fetchmail

last=$(cat $TIMESTAMP_FILE)
echo $(date +%s) >$TIMESTAMP_FILE
write_log "Updated timestamp cache."

modified=$(stat -c '%Y' "$MAIL_INBOX")

# TODO use inotify instead
# compare timestamp
if [ $((modified - last)) -gt 10 ]; then
    # show notification
    notify-send "mail" "You have new mail." -i mutt
    write_log "Sent a notify."
fi

# close log stream
exec 3>&-
```

**P.S.** 关于邮箱服务器的配置

鉴于我的阿里云服务器25端口（SMTP）封禁，正常的邮箱设置也就是通过邮件服务器发送邮件给其他人是行不通的。
所以我就在本机上又搭了一个邮件服务器，没错我又装了一个Postfix。