---
layout: post
title: leetcode Two Sum
date: 2016-09-07
categories: algorithm
tags: [leetcode]
description: 两个数的和。
---

## 题目：
Given an array of integers, return indices of the two numbers such that they add up to a specific target.

## 思路
这一题最直观的思路就是两层for循环，但是这样时间复杂度是O(n^2)。因为题目里告诉了只有唯一解，所有我们可以使用hash来做，具体算法如下：

```go
func twoSum(nums []int, target int) []int {
    m1 := make(map[int]int)
    for index, value := range nums {
        m1[value] = index
    }

    for index, value := range nums {
        complement := target - value
        if v, ok := m1[complement]; ok {
            if m1[complement] != index {
                return []int{index, v}
            }
        }
    }
    return []int{}
}
```