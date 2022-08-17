---
author: "Narcissus"
title: "动态规划-区间DP"
date: "2021-11-19"
description: "动态规划区间DP学习。"
tags: ["动态规划","算法学习"]
categories: ["算法学习"]
---

### 概念

区间 DP 是状态的定义和转移都与区间有关，其中区间用两个端点表示。**状态定义 `dp[i][j] = [i..j] 上原问题的解`。i 变大，j 变小都可以得到更小规模的子问题。**推导状态一般是按照区间长度从短到长推的。

### 大规模问题与常数个小规模问题有关

一般与`dp[i+1][j]`、`dp[i][j-1]`、`dp[i+1][j-1]`有关，`dp[i][j] = f(dp[i+1][j],dp[i][j-1],dp[i+1][j-1])`。

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-19%2013.59.56.png)

时间复杂度和空间复杂度均为O(n^2)。

#### 经典问题

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-19%2014.02.32.png)

考虑**把边界去掉的模式**，可以得到如下状态转移方程：

```go
dp[i][j] = dp[i + 1][j - 1] + 2;                if(s[i] == s[j])
dp[i][j] = max(dp[i + 1][j], dp[i][j - 1]);     if(s[i] != s[j])
```

> 注意遍历方向，根据初始状态选择**反向遍历**

代码如下：

```go
func longestPalindromeSubseq(s string) int {
    // 动态规划 区间动态规划
    n := len(s)
    dp := make([][]int, n)
    for i := 0; i < n; i++ {
        dp[i] = make([]int, n)
        // 初始化
        dp[i][i] = 1
    }
    for i := n-2; i >= 0; i-- {
        for j := i+1; j < n; j++ {
            if s[i] == s[j] {
                dp[i][j] = 2 + dp[i+1][j-1]
            } else {
                dp[i][j] = max(dp[i+1][j], dp[i][j-1])
            }
        }
    }
    return dp[0][n-1]
}

func max(x, y int) int {
    if x > y {
        return x
    }
    return y
}
```

### 大规模问题与 O(n)个小规模问题有关

推导 `dp[i][j]` 时，需要 [i..j] 的所有子区间信息，最常见的形式: `dp[i][j] = g(dp[i][k],...,dp[k][j])`其中`k=i+1...j-1`,g 常见的有 max/min。

#### 经典问题

