---
title: "二分搜索"
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

# 二分搜索

- 在有序数组中判断数字是否存在
- 在有序数组中查找>=num的最左位置
- 在有序数组中查找<=num的最右位置
- 二分搜索不一定只能用在有序数组（峰值问题）
- "二分答案法"
- 二分搜索时间复杂度为O(log n)

## [35. 搜索插入位置](https://leetcode.cn/problems/search-insert-position/)

给定一个排序数组和一个目标值，在数组中找到目标值，并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。

请必须使用时间复杂度为 `O(log n)` 的算法。

**思路**

二分查找来实现，来查找他的值，经过设定每次就算没找到也返回的是它的中值

> 思考：为什么没找到返回的中值也是它的插入位置呢
>
> 因为/2默认是向下取整，也就是说，中值小于目标值的话，此时会记录可能的ans，然后继续往下查找，直到遍历结束，但是如果中值大于目标值，此时不会更新ans，如果一直大于那么ans就是插入位置，如果一开始就是大于目标值的，而且一直大于目标值，则就是最后的一次中值点的位置，最后一次left=right,mid=0,ans=0

```go
func searchInsert(nums []int, target int) int {
    n := len(nums)
    left, right := 0, n - 1
    ans := n
    for left <= right {
        mid := (right - left) >> 1 + left
        if target <= nums[mid] {
            ans = mid
            right = mid - 1
        } else {
            left = mid + 1
        }
    }
    return ans
}

```

## [74. 搜索二维矩阵](https://leetcode.cn/problems/search-a-2d-matrix/)

给你一个满足下述两条属性的 `m x n` 整数矩阵：

- 每行中的整数从左到右按非严格递增顺序排列。
- 每行的第一个整数大于前一行的最后一个整数。

给你一个整数 `target` ，如果 `target` 在矩阵中，返回 `true` ；否则，返回 `false` 。

**思路**

根据二分搜索查找目标值，如果找到直接返回true，如果遍历结束没有找到直接返回false

```go
func searchMatrix(matrix [][]int, target int) bool {
    var l, r , mid int
    for _, nums := range matrix {
        l = 0
        n := len(nums)
        r = n-1
        mid = n
        for l<=r {
            mid = l + (r-l)/2 
            if nums[mid] < target{
                l = mid + 1
            }
            if nums[mid]>target{
                r = mid - 1
            }
            if nums[mid]==target{
                return true
            }
        }
    }
    return false
}
```



## [34. 在排序数组中查找元素的第一个和最后一个位置 ](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array)

给你一个按照非递减顺序排列的整数数组 `nums`，和一个目标值 `target`。请你找出给定目标值在数组中的开始位置和结束位置。

如果数组中不存在目标值 `target`，返回 `[-1, -1]`。

你必须设计并实现时间复杂度为 `O(log n)` 的算法解决此问题

**思路**

可以通过俩次二分算法，一次查找左边界，一次查找右边界

```go
// 用两个边界方法
func searchRange(nums []int, target int) []int {
	// 目标值开始位置：为 target 的左侧边界
	start := findLeftBound(nums, target)
	// 如果开始位置越界 或 目标值不存在，直接返回
	if start == len(nums) || nums[start] != target {
		return []int{-1,-1}
	}
	// 目标值结束位置：为 target 的右侧边界
	end := findRightBound(nums, target)
	return []int{start, end}
}

// 寻找左侧边界的二分查找
func findLeftBound(nums []int, target int) int {
	left, right := 0, len(nums)-1 // note: [left, right]
	for left <= right { // note: 因为 right 是闭区间，所以可以取 =
		mid := left + ((right - left) >> 1) // mid = (left + right) / 2 的优化形式，防止溢出！
		if nums[mid] == target {
			right = mid - 1 // note: 收紧右侧边界以锁定左侧边界
		}else if nums[mid] < target {
			left =  mid + 1
		}else if nums[mid] > target {
			right = mid - 1
		}
	}
	// 返回左侧边界
	return left // note
}

// 寻找右侧边界的二分查找
func findRightBound(nums []int, target int) int {
	left, right := 0, len(nums)-1
	for left <= right {
		mid := left + ((right - left) >> 1)
		if nums[mid] == target {
			left = mid + 1
		}else if nums[mid] < target {
			left =  mid + 1
		}else if nums[mid] > target {
			right = mid - 1
		}
	}
	// 返回右侧边界
	return right
}

```



## [33. 搜索旋转排序数组](https://leetcode.cn/problems/search-in-rotated-sorted-array/description/)

整数数组 `nums` 按升序排列，数组中的值 **互不相同** 。

在传递给函数之前，`nums` 在预先未知的某个下标 `k`（`0 <= k < nums.length`）上进行了 **旋转**，使数组变为 `[nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]`（下标 **从 0 开始** 计数）。例如， `[0,1,2,4,5,6,7]` 在下标 `3` 处经旋转后可能变为 `[4,5,6,7,0,1,2]` 。

给你 **旋转后** 的数组 `nums` 和一个整数 `target` ，如果 `nums` 中存在这个目标值 `target` ，则返回它的下标，否则返回 `-1` 。

你必须设计一个时间复杂度为 `O(log n)` 的算法解决此问题。

**思路**

首先，它肯定一开始是递增的，结尾也是递增的，那就在小于nums[0]大于nums[len-1]不存在，那就在nums[0]~nums[len-1]中就可能存在值小于nums[len-1]，也就是说

- 目标数如果比右边小，那就再跟中值比，如果比中值大，那就在右半部分，如果比中值小，那么中值就是最新右
- 目标数如果比左边大，比中值小，那就在左半边，比中值大，那么中值就是最新左
