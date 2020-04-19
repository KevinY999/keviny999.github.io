---
title: 数据结构八大经典排序的Python实现
date: 2019-07-10 00:12:10
categories: 数据结构
tags: [Algorithm,Python3]
---

八大排序是数据结构里最基本的知识点，也是工作中经常用到的基本算法，因此本文做以整理。

<!-- more -->

## 引入模块

```python
from random import randint, sample
from copy import deepcopy
```



## 冒泡排序

```python
def bubble_sort(lst: list):
    """冒泡排序，时间复杂度O(n^2)，空间复杂度O(1)，稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)
    for i in range(len(ll) - 1, 0, -1):
        for j in range(i):
            if ll[j] > ll[j + 1]:
                ll[j], ll[j + 1] = ll[j + 1], ll[j]
    print(' -  冒泡排序: ', ll)
```



## 选择排序

```python
def choose_sort(lst: list):
    """选择排序，时间复杂度O(n^2)，空间复杂度O(1)，不稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)
    for i in range(len(ll) - 1):
        min_index = i
        for j in range(i + 1, len(ll)):
            if ll[j] < ll[min_index]:
                min_index = j
        ll[i], ll[min_index] = ll[min_index], ll[i]
    print(' -  选择排序: ', ll)
```



## 插入排序

```python
def insert_sort(lst: list):
    """插入排序，时间复杂度O(n^2)，空间复杂度O(1)，稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)
    for i in range(1, len(ll)):
        for j in range(i, 0, -1):
            if ll[j] < ll[j - 1]:
                ll[j], ll[j - 1] = ll[j - 1], ll[j]
            else:
                break
    print(' -  插入排序: ', ll)
```



## 希尔排序

```python
def shell_sort(lst: list):
    """希尔排序，时间复杂度O(nlogn)，空间复杂度O(1)，不稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)
    gap = len(ll) + 1
    while gap > 1:
        gap = gap // 3 + 1
        for i in range(gap, len(ll)):
            temp = ll[i]
            j = i
            while j >= gap and ll[j - gap] > temp:
                ll[j] = ll[j - gap]
                j -= gap
            ll[j] = temp
    print(' -  希尔排序: ', ll)
```



## 归并排序

```python
def merge_sort(lst: list):
    """归并排序，时间复杂度O(nlogn)，空间复杂度O(n)，稳定
    :param lst: List to sort.
    """
    def st(l: list):
        arr = deepcopy(l)
        if len(arr) < 2:
            return arr
        middle = len(arr) // 2
        left = arr[:middle]
        right = arr[middle:]
        return merge(st(left), st(right))

    def merge(left: list, right: list):
        result = []

        while left and right:
            if left[0] <= right[0]:
                result.append(left.pop(0))
            else:
                result.append(right.pop(0))
        if left:
            result.extend(left)
        if right:
            result.extend(right)
        return result

    ll = st(lst)
    print(' -  归并排序: ', ll)
```



## 快速排序

```python
def quick_sort(lst: list):
    """快速排序，时间复杂度O(nlogn)，空间复杂度O(logn)，不稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)

    def qs(l: list):
        if len(l) >= 2:
            base = l[len(l) // 2]
            left, right = list(), list()
            l.remove(base)
            for num in l:
                if num >= base:
                    right.append(num)
                else:
                    left.append(num)
            return qs(left) + [base] + qs(right)
        else:
            return l

    ll = qs(ll)
    print(' -  快速排序: ', ll)
```



## 堆排序

```python
def heap_sort(lst: list):
    """堆排序，时间复杂度O(nlogn)，空间复杂度O(1)，不稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)

    def big_heap(l: list, start: int, end: int):
        while True:
            lchild_index = 2 * start + 1
            rchild_index = 2 * start + 2

            if not lchild_index < end:
                break
            if rchild_index < end:
                max_index = rchild_index if l[rchild_index] > l[lchild_index] else lchild_index
            else:
                max_index = lchild_index
            if l[max_index] > l[start]:
                l[max_index], l[start] = l[start], l[max_index]
                start = max_index
            else:
                break

    length = len(ll)
    for i in range(length // 2 - 1, -1, -1):
        big_heap(ll, i, length)
    for i in range(length - 1, 0, -1):
        ll[i], ll[0] = ll[0], ll[i]
        big_heap(ll, 0, i)

    print(' -    堆排序: ', ll)
```



## 基数排序

```python
def radix_sort(lst: list):
    """基数排序，时间复杂度O(n*k)，空间复杂度O(n+k)，稳定
    :param lst: List to sort.
    """
    ll = deepcopy(lst)

    max_num = ll[0]
    for i in range(1, len(ll)):
        max_num = ll[i] if ll[i] > max_num else max_num
    loop_count, temp = 0, max_num
    while temp:
        loop_count += 1
        temp //= 10
    for c in range(loop_count):
        temp = [(ll[i] // (10 ** c)) % 10 for i in range(len(ll))]
        for i in range(1, len(temp)):
            for j in range(i, 0, -1):
                if ll[j - 1] > ll[j]:
                    ll[j - 1], ll[j] = ll[j], ll[j - 1]
                else:
                    break

    print(' -  基数排序: ', ll)
```



## 测试及结果

```python
if __name__ == '__main__':
    lst = sample(list(set([randint(1, 100) for i in range(20)])), 10)
    print('[+] 原始列表: ', lst, '\n')

    bubble_sort(lst)  # 冒泡排序
    choose_sort(lst)  # 选择排序
    insert_sort(lst)  # 插入排序
    shell_sort(lst)  # 希尔排序
    merge_sort(lst)  # 归并排序
    quick_sort(lst)  # 快速排序
    heap_sort(lst)  # 堆排序
    radix_sort(lst)  # 基数排序
```

![](https://kevin-blog-pic.oss-cn-shenzhen.aliyuncs.com/szjh4.png!unified)