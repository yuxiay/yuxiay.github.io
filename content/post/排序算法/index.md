---
title: "排序算法"
description: 
date: 2024-12-03T22:04:06+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
    - 数据结构与算法
categories:
      - 数据结构与算法
---

# 排序算法

## 选择排序

简要说明：

​	在i~n-1范围上，找到最小值并放在i位置，然后i+1~n-1范围上继续，即遍历数组依次找到最小值

代码实现：

```go
func SelectSort(args []int) []int {
	n := len(args)
	for i := 0; i < n-1; i++ {
		minIndex := i // 初始化最小值为当前起始位置
		for j := i + 1; j < n; j++ {
			if args[j] < args[minIndex] {
				minIndex = j // 更新最小值索引
			}
		}
		// 交换当前位置与最小值位置
		args[i], args[minIndex] = args[minIndex], args[i]
	}
	return args
}

```



## 冒泡排序

简要说明：

​	在0~n的范围上，相邻位置最大的数进行俩俩交换，然后在0~n-1的范围上继续

代码实现：

```go
func BubbleSort(args []int) []int {
	n := len(args)
	for i := 0; i < n-1; i++ {
		for j := 0; j < n-1-i; j++ {
			if args[j] > args[j+1] {
				args[j], args[j+1] = args[j+1], args[j]
			}
		}
	}
	return args
}

```



## 插入排序

简要说明：

​	在0~i的位置上已经有序，新来的数从右往左，滑倒不能再小的数进行插入，与摸牌类似

代码实现：

```go
func InsertionSort(arr []int) { // Implementation of Insertion Sort
    for i := 1; i < len(arr); i++ {
        key := arr[i]
        j := i - 1
        for j >= 0 && arr[j] > key {
            arr[j+1] = arr[j]
            j--
        }
        arr[j+1] = key
    }
}
```



