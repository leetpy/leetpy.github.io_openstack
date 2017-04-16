---
layout: post
title:  "python with语句"
date:   2017-03-11 16:05:14 +0800
categories: python
tags: [python]
location: Nanjing, China
description: 介绍了python测试框架nose的基本用法.
---

# python with语句

with 语句是用来替代try-except-finall语句的，使代码更加简洁。with语句的基本语法如下：

```python
with context [as var]:
    with_suite
```

context表达式返回的是一个对象，var用来保存context返回的对象，可以是单个返回值，也可以是元组。

with 语句的实质是上下文管理。

1. 上下文管理协议：包含方法：`__enter__()`和`__exit__()`，支持该协议的对象要实现这两个方法。
2. 上下文管理器：定义执行with语句时要建立的运行时上下文，负责执行with语句块上下文中的进入与退出操作。

with虽然可以