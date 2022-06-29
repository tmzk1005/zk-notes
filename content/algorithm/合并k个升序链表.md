---
title: "合并k个升序链表"
date: 2021-03-01T22:00:00+08:00
draft: false
toc: true
---

# 题述

https://leetcode-cn.com/problems/merge-k-sorted-lists/

给你一个链表数组，每个链表都已经按升序排列。请你将所有链表合并到一个升序链表中，返回合并后的链表。

**示例1**

```
输入：lists = [[1,4,5],[1,3,4],[2,6]]
输出：[1,1,2,3,4,4,5,6]
解释：链表数组如下：
[
  1->4->5,
  1->3->4,
  2->6
]
将它们合并到一个有序链表中得到。
1->1->2->3->4->4->5->6
```

**示例2**

```
输入：lists = []
输出：[]
```

**示例3**

```java
输入：lists = [[]]
输出：[]
```

**提示**

```
k == lists.length
0 <= k <= 10^4
0 <= lists[i].length <= 500
-10^4 <= lists[i][j] <= 10^4
lists[i] 按 升序 排列
lists[i].length 的总和不超过 10^4
```

# 题解

此题在leecode上是hard级别的，但是其实比较简单。

写一个子函数，把各个链表的头中最下的节点分离出来，分离出来的节点接到已经分离出的链表后面就可以了，剩下的链表数组其中的一个链表长度就减少了1，剩下的问题是同样的问题，因此循环一次就可以解决了。

## 代码

```java
public class Solution {

    public ListNode mergeKLists(ListNode[] lists) {
        ListNode head = new ListNode(0);
        ListNode tail = head;
        ListNode splitNode;
        while ((splitNode = splitMinNode(lists)) != null) {
            tail.next = splitNode;
            tail = splitNode;
        }
        return head.next;
    }

    public ListNode splitMinNode(ListNode[] lists) {
        int index = -1;
        int min = Integer.MAX_VALUE;
        for (int k = 0; k < lists.length; ++k) {
            if (lists[k] == null) {
                continue;
            }
            if (lists[k].val < min) {
                min = lists[k].val;
                index = k;
            }
        }
        if (index == -1) {
            return null;
        }
        ListNode result = lists[index];
        lists[index] = result.next;
        return result;
    }

}

class ListNode {
    int val;
    ListNode next;

    ListNode(int val) {
        this.val = val;
    }
}
```