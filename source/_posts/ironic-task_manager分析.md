---
layout: post
title: task_manager
date: 2016-09-07
categories: algorithm
tags: [leetcode]
description: ironic task_manager介绍
---

## task_manager分析

TaskManager暴露如下属性和资源：

- task.context

  ​	传给TaskManager的context

- task.shared

  ​	如果Node locked返回False，否则返回True

- task.node

- task.ports

- task.volume_connectors

- task.volume_targets

- task.driver



```python
def require_exclusive_lock(f):

    @six.wraps(f)
    def wrapper(*args, **kwargs):
        # 这里这么写是测试的时候会用到Mock
        if len(args) > 1:
            task = args[1] if isinstance(args[1], TaskManager) else args[0]
        else:
            task = args[0]
        if task.shared:
            raise exception.ExclusiveLockRequired()
            
        task.context.ensure_thread_contain_context()
        return f(*args, **kwargs)
    return wrapper
```



创建TaskManager

```python
def acquire(context, node_id, shared=False, driver_name=None,
            purpose='unspecified action'):

    # NOTE(lintan): This is a workaround to set the context of periodic tasks.
    context.ensure_thread_contain_context()
    return TaskManager(context, node_id, shared=shared,
                       driver_name=driver_name, purpose=purpose)
```

