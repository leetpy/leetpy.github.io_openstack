---
layout: post
title: 常用shell
date: 2017-03-12
categories: shell
tags: [shell]
description: 常用shell命令。
---

## 获取脚本所在路径
```shell
export SCRIPT_HOME=$(dirname $(readlink -f $0))
```

## 解压压缩
```shell
# tar.gz
tar -zxvf filename.tar.gz 
tar -zcvf filename.tar.gz target

# tar
tar -xvf filename.tar 
tar -cvf filename.tar target

# zip
zip -r ./*
unzip filename.zip 
```

## lvm
### 扩容
```shell
# 先用fdisk创建新分区，gpt分区使用gdisk命令

# 刷新分区表
partprobe

#　创建pv
pvcreate /dev/sda5

# vg扩容
vgextend vg_sys /dev/sda5

#　lv扩容
lvextend -l +100%FREE  /dev/mapper/vg_sys-lv_root

vgchange -ay vg_sys
resize2fs /dev/mapper/vg_sys-lv_root
```
