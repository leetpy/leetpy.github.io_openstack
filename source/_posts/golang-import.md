---
layout: post
title: golang import
date: 2016-07-05
categories: golang
tags: [golang]
description: golang import操作。
---

# golang import操作

## import系统包
在导入golang自带或者是我们安装的第三方库时，这些库都是直接以名字开头的，例如"fmt", "container/list"。这些直接import即可。这个时候golang是从Go Tree（跟环境变量配置有关）里去寻找这些包。
 
    // 可以使用
    import "fmt"
    import "os"
    import "fmt"; import "os"
    
    // 推荐方法
    import (
        "fmt"
        "os"
    )
    
## immport自定义模块
在我们的project比较大时，我们常常会自己定义一些模块，在导入这些包时，我们需要使用绝对路径或者相对路径。

    // 导入自定义包
    import ./model
    
## 其它操作
有时候我们在看一些开源代码或者框架介绍的时候，会在import的包名前面加一个“.”或着"_"。

* "."的作用是在你使用这个包的函数时，可以省略报名

    // eg:
    import "fmt"
    Println("hello")
    
* "_"的作用是在导入包的时候会之行包的init函数

## 使用别名
    import (
        f "fmt"
    )
    
    func main() {
        f.Println("Hello, world")
    }