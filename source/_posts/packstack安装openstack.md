---
layout: post
title:  "packstack安装openstack"
date:   2017-04-02 16:05:14 +0800
categories: openstack
tags: [packstack]
location: Nanjing, China
description: packstack安装openstack.
---

## 环境准备

这里我们以CentOS7为例，安装openstack ocata版本，其它版本安装方法类似。packstack目前对NetworkManager还不支持，我们修改下配置：
```shell
systemctl disable firewalld
systemctl stop firewalld
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network
```

### 安装

```shell
# 添加packstack yum源
yum -y install centos-release-openstack-ocata

# 安装openstack-packstack
yum -y install openstack-packstack

# 生成answer-file
packstack --gen-answer-file=filename
```

### 安装openstack

如果你安装的是ocata版本，这里packstack有电小bug，有几个文件需要修改一下，参考：

[问题描述](https://www.redhat.com/archives/rdo-list/2017-March/msg00011.html)

两个bug的review链接：

- [https://review.openstack.org/#/c/440258/](https://review.openstack.org/#/c/440258/)
- [https://review.openstack.org/442551](https://review.openstack.org/442551)

照着review提交的内容修改一下就可以了。接着使用下面的命令安装openstack。

```shell
packstack --answer-file=filename
```

