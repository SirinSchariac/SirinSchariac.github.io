---
layout: 	post  				   
title: 		Golang Sclices     				
subtitle:	append slices的细节
date:       2025-09-28 				
author:     Sirin 						
header-img: img/train.jpg
catalog: 	true 				
tags:						
    - Go
---

<font color = skyblue font size = 5>关于Golang中的slices append操作</font>

在Leetcode回溯相关的题目中(e.g. [131. 分割回文串 - 力扣（LeetCode）](https://leetcode.cn/problems/palindrome-partitioning/description/?envType=study-plan-v2&envId=top-100-liked))，涉及到对于回溯过程中得到的多种方案的答案合并问题，采用的是`ans = append(ans, append([]string(nil), splits...))`，这就引出一个问题，为什么不直接采用`ans = append(ans, splits)`?

#### 为什么不能直接 `append(ans, splits)`

直接使用 `append(ans, splits)` 的问题是**共享底层数组**，这会导致：

```go
ans := [][]string{}
splits := []string{"a", "b", "c"}

// 错误的方式 - 共享底层数组
ans = append(ans, splits)

// 如果后续修改 splits，ans 中的对应元素也会被修改！
splits[0] = "modified"
fmt.Println(ans[0][0]) // 输出 "modified"，这不是我们想要的
```

#### 为什么需要 `append([]string(nil), splits...)`

这种写法实际上是创建了一个**新的底层数组**：

```go
// 正确的方式 - 创建副本
ans = append(ans, append([]string(nil), splits...))

// 现在修改 splits 不会影响 ans
splits[0] = "modified"
fmt.Println(ans[0][0]) // 仍然输出 "a"，不受影响
```

#### 等价写法

`append([]string(nil), splits...)` 等价于：

```go
// 方式1：使用 make 和 copy
newSlice := make([]string, len(splits))
copy(newSlice, splits)
ans = append(ans, newSlice)

// 方式2：使用字面量展开（推荐，更简洁）
ans = append(ans, append([]string{}, splits...))

// 方式3：循环追加（不推荐，性能较差）
newSlice := []string{}
for _, s := range splits {
    newSlice = append(newSlice, s)
}
ans = append(ans, newSlice)
```

这种模式在**回溯算法**中特别常见：

```go
func backtrack(path []string, result *[][]string) {
    if condition {
        // 必须创建 path 的副本，否则后续修改会影响结果
        *result = append(*result, append([]string{}, path...))
        return
    }
    
    for _, choice := range choices {
        path = append(path, choice)  // 做出选择
        backtrack(path, result)
        path = path[:len(path)-1]    // 撤销选择
    }
}
```

#### 总结

- **直接append**：共享底层数组，修改原slice会影响结果
- **创建副本再append**：完全独立，互不影响
- `append([]string(nil), splits...)` 是创建副本的简洁写法

这种写法确保了二维slice中每个一维slice都是独立的，避免了意外的数据共享问题。

---

#### Plus

> `splits...` 和 `splits` 的区别?

`append(dst, splits...)` 和 `append(dst, splits)` 是完全不同的东西：

- `append(dst, splits...)` 表示把 `splits` **里的元素逐一拷贝**到 `dst` 末尾。
- `append(dst, splits)` 会报错，因为 **`append` 第二个参数必须是元素类型或同类型 slice 的展开**，不接受直接传一个 slice 变量（除非目标类型是 `[][]string`）。

举个例子：

```go
a := []string{"x", "y"}
b := []string{"a", "b"}
c := append(a, b...) // c = {"x","y","a","b"}
```

如果你写 `append(a, b)`，编译器会报错：
 `cannot use b (type []string) as type string in append`

