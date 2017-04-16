---
layout: post
title:  "python测试框架nose使用"
date:   2017-03-11 16:05:14 +0800
categories: python
tags: [python]
location: Nanjing, China
description: 介绍了python测试框架nose的基本用法.
---

# 使用nose进行单元测试

## nose安装
```python
pip install nose
```

## Example
例如我们在multiply.py文件中有如下一段代码:

```python
def multiply(x, y):
    return x * y
```

为了测试上面的代码，我们添加test_multiply.py，编写如下内容：

```python
from multiply import multiply


def test_number_3_4():
    assert multiply(3, 4) == 12


def test_strings_a_3():
    assert multiply('a', 3) == 'aaa'
```
运行`nosetests`,打印结果如下：

```shell
yl@lee:~/code/py/project$ nosetests
..
----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK
```
如果要查看详细信息我们可以添加-v参数：

```shell
yl@lee:~/code/py/project$ nosetests -v
multiply_test.test_number_3_4 ... ok
multiply_test.test_strings_a_3 ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK
```

nose会自动匹配test用例，匹配规则是：满足`(?:^|[b_.-])[Tt]est`的类，函数，目录，方法。

## nose fixtures
在测试一组用例的时候，有些初始化或结束代码是通用的，我们可以把这部分代码提取出来，放到setup和teardown中。

- 在module生效，使用setup_module/teardown_module
- 在class生效，使用setup_class/teardown_class，并添加`@classmethod`装饰器
- function使用setup_function/teardown_function,并添加`@with_setup`装饰器

**备注**
- setup_module(): 在文件中最早执行
- teardown_module(): 在文件中最后执行
- setup()在类所有方之前执行
- teardown()在类所有方法之后执行
- setup_class()在类每个方法开始时执行
- teardown_class()在类每个方法最后执行

具体例子如下：

```python
from nose import with_setup # optional
from unnecessary_math import multiply

def setup_module(module):
    print ("") # this is to get a newline after the dots
    print ("setup_module before anything in this file")

def teardown_module(module):
    print ("teardown_module after everything in this file")

def my_setup_function():
    print ("my_setup_function")

def my_teardown_function():
    print ("my_teardown_function")

@with_setup(my_setup_function, my_teardown_function)
def test_numbers_3_4():
    print 'test_numbers_3_4  <============================ actual test code'
    assert multiply(3,4) == 12

@with_setup(my_setup_function, my_teardown_function)
def test_strings_a_3():
    print 'test_strings_a_3  <============================ actual test code'
    assert multiply('a',3) == 'aaa'


class TestUM:

    def setup(self):
        print ("TestUM:setup() before each test method")

    def teardown(self):
        print ("TestUM:teardown() after each test method")

    @classmethod
    def setup_class(cls):
        print ("setup_class() before any methods in this class")

    @classmethod
    def teardown_class(cls):
        print ("teardown_class() after any methods in this class")

    def test_numbers_5_6(self):
        print 'test_numbers_5_6()  <============================ actual test code'
        assert multiply(5,6) == 30

    def test_strings_b_2(self):
        print 'test_strings_b_2()  <============================ actual test code'
        assert multiply('b',2) == 'bb'
```

默认情况下nose不会打印程序的输出，加上`-s`参数可以打印输出

```shell
yl@lee:~/code/py/project$ nosetests -v -s

setup_module before anything in this file
setup_class() before any methods in this class
multiply_test.TestUM.test_numbers_5_6 ... TestUM:setup() before each test method
test_numbers_5_6()  <============================ actual test code
TestUM:teardown() after each test method
ok
multiply_test.TestUM.test_strings_b_2 ... TestUM:setup() before each test method
test_strings_b_2()  <============================ actual test code
TestUM:teardown() after each test method
ok
teardown_class() after any methods in this class
multiply_test.test_numbers_3_4 ... my_setup_function
test_numbers_3_4  <============================ actual test code
my_teardown_function
ok
multiply_test.test_strings_a_3 ... my_setup_function
test_strings_a_3  <============================ actual test code
my_teardown_function
ok
teardown_module after everything in this file

----------------------------------------------------------------------
Ran 4 tests in 0.002s

OK
```

## 使用nose assert语句
```python
from nose.tools import assert_equals

def test_numbers_3_4():
    assert_equals(multiply(3,4), 12)
```

常用assert语句如下：

- assert_almost_equal(first, second, places=7, msg=None)
- assert_almost_equals
- assert_not_almost_equal
- assert_not_almost_equals
- assert_equal(first, second, place=7, msg=None)
- assert_equals
- assert_false
- assert_true
- assert_not_equal
- assert_not_equals
- eq_
- ok_

## 异常处理
有时候我们会在程序的某些地方抛异常，对于这种情况，需要使用@raises装饰器处理。

```python
def play():
    sys.exit(1)

from nose.tools import raises
@raises(SystemExit)
def test_play_except():
    play()
```

## 常用参数
- nosetests -v： debug模式，看到具体执行情况，推荐使用；
- nose会捕获标准输出，程序中的print不会打印到出来，使用nosetests -s可以打开output输出；
- 默认nosetests会执行所有的test case，如果想单独执行一个case，执行nosetests --tests后跟要测试的文件；
- nosetests --pdb-failures:失败时，立马调试。这个选项很赞，可以看到失败时的及时环境；
- nosetests --collect-only -v: 不运行程序，只是搜集并输出各个case的名称；
- nosetests -x:一旦case失败，立即停止，不执行后续case;
- nosetestx -failed:只执行上一轮失败的case;

## 命名规范
- module使用 'test_'开头
- fucntion使用 'tets_'开头
- class使用 'Test'开头
- method使用'test_'开头
- 测试代码的package里有'init.py'

## 获取nose返回值
- shell
在shell下执行时，如果全部用例都通过，则返回0，有failed或error则返回1。

- python
在python代码中调用nose.run()函数，如果全部用例都通过，返回True，有failed或error返回False。

默认情况下，nose会屏蔽所有输出，如果要打开调试信息可以通过如下方式：

```python
result = nose.run(defaultTest="", argv=['', '--nocapture'])
```