# 压力测试 chiab

### 介绍

[chiab](https://github.com/langwan/chiab) 是一款类似于ab的简单压力测试工具，使用go语言编写的第三方库。ab是一个独立的命令行工具，chiab是嵌入在代码当中执行的函数。chiab的代码仅仅只有100行左右。

chiab可以生成明确的报告文件，方便对测试结果进行管理和分发。

### 测试http请求

```go
func TestGet(t *testing.T) {
   var concurrency int64 = 20
   var requests int64 = 10000
   RequestStart(concurrency, 60*time.Second)
   Run(func(id int64) bool {
      _, err := Get(id, "http://127.0.0.1:8100/profile", nil, "")
      if err != nil {
         return false
      } else {
         return true
      }
   }, concurrency, requests, "测试HTTP服务", false)
}
```

### 测试代码段

```go
func TestRun(t *testing.T) {
   var concurrency int64 = 20
   var requests int64 = 10000
   RequestStart(concurrency, 60*time.Second)
   Run(func(id int64) bool {
      return true
   }, concurrency, requests, "测试函数执行效率", true)
}
```

### Run方法

参数：

* **handler** 被测试的代码段
* **concurrency** 并发数
* **requests** 请求数
* **title** 报告标题
* **save** 报告是否存为文件

### 测试报告

文件名：测试函数执行效率\_2022-12-05 11:39.txt

```
Complete requests:       10000
Requests per second:     3443477.554208 [#/sec]
Time taken for tests:    2.904041ms
Failed requests:         0
p90:                     84ns
max time:                86.833µs
min time:                0s
Concurrency Level:       20
```

### 实现

#### 发令枪
