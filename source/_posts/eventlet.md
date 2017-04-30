---
title: eventlet 使用
date: 2017-04-19 19:44:39
categories: python
tags: [I/O]
---

熟悉openstack 的朋友可能经常见到eventlet这个库。eventlet是一个python 并发网络库，eventlet采用协程实现，具有如下优点：
- 使用epoll, kqueue，libevent 提供高性能无阻塞I/O
- 采用协程，无阻塞I/O，并发高，消耗资源少
- 事件分发

## 协程化
在使用eventlet时，比较难的是协程化import的库，很多python 库编写的时候就不支持协程，但是eventlet提供了如下几种方法，来协程化其它库。
- 标准库
eg: 
```python
from eventlet.green import socket
from eventlet.green import threading
from eventlet.green import asyncore
```

- 非标准库: import_patched
eg: 
```python
import eventlet

eventlet.import_patched('BaseHTTPServer')
```

- 非标准库：monkey_patch
虽然eventlet.import_patched()可以协程化非标准库，但是对于运行时导入的库是无法进程协程化的，使用eventlet.monkey_patch()则可以动态导入，但是会出现幻数。
```python
import eventlet

eventlet.monkey_patch()
```

![eventlet 常用方法](/img/eventlet.png)

下面是使用 eventlet 下载http目录的代码：
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import eventlet
from bs4 import BeautifulSoup
import os
import time
from urlparse import urlparse


requests = eventlet.import_patched('requests')

def ensure_tree(path):
    if os.path.isdir(path):
        return
    try:
        os.makedirs(path)
    except:
        pass


def get_page_links(parent_url, page, jobs):
    soup = BeautifulSoup(page)
    hrefs = soup.findAll('a')
    for i in hrefs:
        lk = i.attrs['href']
        if not lk.startswith('..') and not lk == './':
            if lk.startswith('http') or lk.startswith('https'):
                jobs.put(lk)
            else:
                jobs.put(os.path.join(parent_url, lk))


def download(url, jobs):
    parts = urlparse(url)
    output = parts.path
    if output.startswith('/'):
        output = output[1:]
    if os.path.exists(output):
        return True, url
    try:
        r = requests.get(url, stream=True)
        content_type = r.headers.get('Content-Type')
        if content_type.lower() == 'text/html':
            links = get_page_links(url, r.content, jobs)
        else:
            path = os.path.dirname(output)
            ensure_tree(path)
            with open(output, 'wb') as hanle:
                if not r.ok:
                    print "download fail"
    
                for block in r.iter_content(1024):
                    hanle.write(block)
            
        # print "[x] %s" % output
        return True, url
    except:
        return False, url
    

def add_jobs(filename, jobs):
    with open(filename) as fp:
        lines = fp.readlines()
        for url in lines:
            jobs.put(url.strip())


def product(jobs):
    pool = eventlet.GreenPool()
    while True:
        while not jobs.empty():
            url = jobs.get()
            pool.spawn_n(download, url, jobs)
        pool.waitall()
        if jobs.empty():
            break

if __name__ == '__main__':
    t1 = time.time()
    jobs = eventlet.Queue()
    
    jobs = eventlet.Queue()
    add_jobs('urls.txt', jobs)
    product(jobs)
    t2 = time.time()
    print t2 - t1
```