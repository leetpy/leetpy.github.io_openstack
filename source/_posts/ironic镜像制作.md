---
layout: post
title: ipa镜像制作
date: 2017-04-04
categories: openstack
tags: [ipa]
description: ipa镜像制作
---

## 镜像介绍
ironic部署需要两组镜像，一组是deploy镜像，另一组是user镜像，其中deploy镜像包括ipa服务。user镜像又分为whole disk image和partition image，其中whole disk image 是带bootloader和分区表的。

## 工具
制作ironic deploy镜像其实就是在普通镜像中添加一个ipa服务，用来裸机和ironic通信。官方推荐制作镜像的工具有两个，
分别是CoreOS tools和disk-image-builder

具体链接如下：[building deploy ramdisk](https://docs.openstack.org/project-install-guide/baremetal/ocata/deploy-ramdisk.html)

### CoreOS
如果你不想自己制作镜像，可以从[CoreOS img](http://tarballs.openstack.org/ironic-python-agent/coreos/files/)下载

可以使用官方文档中的方法自己制作：
```shell
cd ironic-python-agent/imagebuild/coreos
sudo systemctl start docker
sudo make
sudo make iso
```

### disk-image-builder
这个工具可以自定义，通过element的方式添加很多特性。
```shell
# deploy image
disk-image-create ironic-agent ubuntu -o ironic-deploy

# whole disk image
disk-image-create ubuntu vm dhcp-all-interfacec -o my-image

# partation image
disk-image-create ubuntu baremetal dhcp-all-interfacec -o my-image
```

## 密码问题
如果是CoreOS镜像，修改`/etc/ironic/ironic.conf`，在pxe_append_params参数后面添加：sshkey="ssh-rsa AAAA..."，
然后重启ironic-conductor服务，这时你就可以使用ssh core@<ip-address-of-node>登录了。这里的sshkey最好写conductor节点的。

如果是disk-image-builder镜像，有两种方法添加密码：
- devuser
在制作镜像的时候，通过环境变量注入密码：
```shell
export DIB_DEV_USER_USERNAME=username
export DIB_DEV_USER_PWDLESS_SUDO=yes
export DIB_DEV_USER_AUTHORIZED_KEYS=$HOME/.ssh/id_rsa.pub
disk-image-create -o /path/to/custom-ipa debian ironic-agent devuser
```

- dynamic-login
这种方式是官方推荐的工具，因为devuser方式不太安全，所有人都可以登录。
dynamic-login的思想是在ipa镜像里起一个dynamic-login服务，然后通过cmd参数把ssk public key或者密码在上电的时候带进来，这样就可以顺利登录系统了。
