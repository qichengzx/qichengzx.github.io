title: LeetCode 561.Array Partition I(数组分区 1) - Go实现
categories: golang
date: 2017-08-13 10:14:49
tags: [go,leetcode]

---

题目地址:[561. Array Partition I](https://leetcode.com/problems/array-partition-i/description/)

题目描述：

Given an array of 2n integers, your task is to group these integers into n pairs of integer, say (a1, b1), (a2, b2), ..., (an, bn) which makes sum of min(ai, bi) for all i from 1 to n as large as possible.

###### Example 1:

```
Input: [1,4,3,2]

Output: 4

Explanation: n is 2, and the maximum sum of pairs is 4 = min(1, 2) + min(3, 4).
```

###### Note:

​	n is a positive integer, which is in the range of [1, 10000].
​	All the integers in the array will be in the range of [-10000, 10000].



题目大意：

给定一个长度为2n的数组，要把它分成n个分组，即每组有两个数，返回每组中最小值的总和，使和最大。

理解了大意就知道思路了，又看了下论坛里的[算法分析](https://discuss.leetcode.com/topic/87206/java-solution-sorting-and-rough-proof-of-algorithm)

解决方案基本就是先按从小到大排序，这样相邻的数字是最接近的，然后再分成两两一组，取每组中的第一个数相加即可。

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	nums := []int{4, 5, 6, 1}
	n := arrayPairSum(nums)
	fmt.Println(n)
}

func arrayPairSum(nums []int) int {
	sort.Ints(nums)
	sum := 0
	length := len(nums)
	for i := 0; i < length; i += 2 {
		sum += nums[i]
	}
	return sum
}
```

在线查看结果:[The Go Playground](https://play.golang.org/p/2tkZyayXUB)

其实这里主要用到了Go的[sort包给int数组排序](https://golang.org/pkg/sort/#Ints)。排序后遍历数组，每次递增2就可以了。