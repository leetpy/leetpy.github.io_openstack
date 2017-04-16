---
layout: post
title:  "mac mysql启动配置"
date:   2017-03-11 16:05:14 +0800
categories: mac
tags: [mysql, mac]
location: Nanjing, China
description: mac mysql启动配置.
---

mac下使用mysql有点蛋疼，每次都要找命令。可能不同版本或者安装方式mysql的位置不太一样，可以使用`locate mysql.server`查找一下。

```shell
# start
sudo /usr/local/bin/mysql.server start

# stop
sudo /usr/local/bin/mysql.server stop

# restart
sudo /usr/local/bin/mysql.server restart
```

如果不想每次都敲这么一长串的命令，可以使用alias。在~/.bash_profile中添加：

```shell
alias mysqlstart='sudo /usr/local/bin/mysql.server start'
alias mysqlstop='sudo /usr/local/bin/mysql.server stop'
```