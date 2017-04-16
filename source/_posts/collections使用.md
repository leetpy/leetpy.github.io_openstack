---
layout: post
title: python collections模块
date: 2017-04-04
categories: python
tags: [python]
description: python collections模块介绍
---
## namedtuple

namedtuple

```python
from collections import namedtupule

Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)

print p.x
print p.y
```



## deque

deque是双向链表实现的，相比list具有更高的插入和删除效率。

```python
>>> from collections import deque

>>> q = deque(['a', 'b', 'c'])
>>> q.append('x')
>>> q.appendleft('y')
>>> q
deque(['y', 'a', 'b', 'c', 'x'])
```

## defaultdict

在使用普通的dict类型时，我们在取值时需要判断key是否存在，如果我们希望当一个key不存在时返回默认值，我们可以使用defaultdict。

```python
>>> from collections import defaultdict

>>> dd = defaultdict(lamba: 'No key')
>>> dd['a'] = 'A'
>>> print dd['a']
'A'
>>> print dd['b']
'No key'
```

## OrderedDict

OrderedDIct见名知意，就是key是有序的。

```python
>>> from collections import OrderedDict
>>> od = OrderDict([('z', 1), ('y', 2), ('x', 3)])
>>> od.keys()
['z', 'y', 'x']
```

需要注意的是，OrderedDict key是按插入的顺序排列的，而不是字符串大小。

## Counter

 Counter是一个简单的计数器。

```python
>>> from collections import Counter
>>> c = Counter()
>>> for ch in "aaaabb":
...		c[ch] += 1
>>> c
Counter({'a': 4, 'b': 2})
```

