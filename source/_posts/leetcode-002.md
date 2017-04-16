---
layout: post
title: leetcode Add Two Numbers
date: 2016-09-11
categories: algorithm
tags: [leetcode]
description: 两个数的和。
---

## 题目：
You are given two linked lists representing two non-negative numbers.The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.
    
```
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
```

## 思路
这一题比较简单，因为链表的第一位就是数字的个位，依次将每一位相加即可以，需要考虑进位的情况。

---
**编程时注意以下几点：**

1. 每次计算完，记得l = l.Next
2. 使用head指针，最后返回head.Next
3. 所有加完了要判断进位大小

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    sumNode := &ListNode{0, nil}
    p := sumNode
    var carry int = 0

    for {
        if l1 != nil && l2 != nil {
            sum := l1.Val + l2.Val + carry
            p.Next = &ListNode{sum % 10, nil}
            p = p.Next
            l1 = l1.Next
            l2 = l2.Next
            carry = sum / 10
        } else {
            break
        }
    }

    for {
        if l1 != nil {
            sum := l1.Val + carry
            p.Next = &ListNode{sum % 10, nil}
            p = p.Next
            l1 = l1.Next
            carry = sum / 10
        } else {
            break
        }
    }

    for {
        if l2 != nil {
            sum := l2.Val + carry
            p.Next = &ListNode{sum % 10, nil}
            p = p.Next
            l2 = l2.Next
            carry = sum / 10
        } else {
            break
        }
    }

    if carry != 0 {
        p.Next = &ListNode{carry, nil}
    }
    return sumNode.Next
}
```