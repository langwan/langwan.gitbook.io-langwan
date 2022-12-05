# 压力测试 chiab

### 介绍

[chiab](https://github.com/langwan/chiab) 是一款类似于ab的简单压力测试工具，使用go语言编写的第三方库。ab是一个独立的命令行工具，chiab是嵌入在代码当中执行的函数。chiab的代码仅仅只有100行左右。

### 为什么要写chiab？

1. 生成准确测试时间的标准报告文件
2. 在go语言中直接调用更方便

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

#### 定义Request

```go
type request struct {
   ok      bool
   runtime time.Duration
}
```

#### 任务分配

给每个并发平均分配任务，如果不能平均最后一个协程会分配的多一些。

```go
rem := requests % concurrency
requestsPerWorker := (requests - rem) / concurrency
```

#### 启动携程池

```go
go func() {
   var i int64 = 0
   for ; i < concurrency; i++ {
      if i == concurrency-1 {
         requestsPerWorkers += rem
      }
      go worker(i, handler, requestsPerWorkers)
   }
}()
```

#### 发令枪

让所有的协程先预热跑起来，等待发令枪，发令枪响，所有协程开始跑被分配的工作单元。

```go
for readyCount < concurrency {
   select {
   case <-chReady:
      readyCount++
   }
}

var i int64 = 0
for ; i < concurrency; i++ {
   chGun <- struct{}{}
}
```

#### 日志写入文件

<pre class="language-go"><code class="lang-go"><strong>if save {
</strong>   logFile, err := os.OpenFile(logName, os.O_RDWR|os.O_CREATE, 0644)
   mw := io.MultiWriter(os.Stdout, logFile)
   if err != nil {
      panic(err)
   }
   log.SetOutput(mw)
}
</code></pre>

#### 等待每一个任务结束

```go
for len(totalRequests) < int(requests) {
   req, ok := <-chCompleted
   if ok {
      totalRequests = append(totalRequests, req)
   }
}
```

#### 报告

```go
func reporting(concurrency int64, requests int64) {
   m := reportingItem{}
   succeedCount := 0
   for _, request := range totalRequests {
      if request.ok {
         succeedCount++
      }
   }

   sort.SliceStable(totalRequests, func(i, j int) bool {
      return totalRequests[i].runtime < (totalRequests)[j].runtime
   })
   m.minRuntime = totalRequests[0].runtime
   m.maxRuntime = totalRequests[len(totalRequests)-1].runtime

   n90 := int(math.Floor(90.0 / 100.0 * float64(len(totalRequests))))

   m.p90 = totalRequests[n90].runtime
   m.qps = float64(len(totalRequests)) / runtime.Seconds()
   log.Printf("%-25s%d\n", "Complete requests:", len(totalRequests))
   log.Printf("%-25s%f [#/sec]\n", "Requests per second:", m.qps)
   log.Printf("%-25s%s\n", "Time taken for tests:", runtime.String())
   log.Printf("%-25s%d\n", "Failed requests:", len(totalRequests)-succeedCount)
   log.Printf("%-25s%s\n", "P90:", m.p90.String())
   log.Printf("%-25s%s\n", "Max time:", m.maxRuntime.String())
   log.Printf("%-25s%s\n", "Min time:", m.minRuntime.String())
   log.Printf("%-25s%d\n", "Concurrency Level:", concurrency)
}
```

#### P90

百分之90的用户访问速度

```go
sort.SliceStable(totalRequests, func(i, j int) bool {
   return totalRequests[i].runtime < (totalRequests)[j].runtime
})
m.minRuntime = totalRequests[0].runtime
m.maxRuntime = totalRequests[len(totalRequests)-1].runtime

n90 := int(math.Floor(90.0 / 100.0 * float64(len(totalRequests))))

m.p90 = totalRequests[n90].runtime
```

### 地址

[https://github.com/langwan/chiab](https://github.com/langwan/chiab)
