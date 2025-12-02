---
title: Go pprof 使用与内存分析技术笔记
date: 2025-12-02 20:54:00
tag: #技术
---
##  1. pprof 是什么

pprof 是 Go 内置的性能分析工具，支持分析 CPU、内存（Heap）、Goroutine、Block、Mutex 等指标。
常用场景：
* 内存暴涨、OOM
* Goroutine 泄漏
* CPU 使用异常飙升
* IO / 锁争用

## 2. 内存相关的两种 profile：heap vs allocs
| profile | 说明                                      | 场景                             |
| ------- | ----------------------------------------- | -------------------------------- |
| heap    | 采集当前时刻内存占用快照（in-use memory） | 查内存泄漏 / 常驻内存过高        |
| allocs  | 累计分配（所有分配次数和大小）            | 查高频分配热点，不一定仍在内存中 |

### 应该什么时候抓取？
* 排查任务执行中内存增长：任务峰值时抓 heap
* 排查任务结束后仍占用高内存：结束后抓 heap（判断泄漏）
* 排查频繁分配导致 GC 压力大：抓 allocs

> heap = 当前存活对象
> 
> allocs = 历史分配对象（包括已释放的）

## 3. 获取 profile

常见方式：

### 3.1 程序内启动 HTTP pprof
```go
import _ "net/http/pprof"
go func() {
    log.Println(http.ListenAndServe(":6060", nil))
}()
```

访问：
```
http://localhost:6060/debug/pprof/heap
http://localhost:6060/debug/pprof/allocs
```

### 3.2 生成本地文件
```bash
curl -o heap.out http://localhost:6060/debug/pprof/heap
go tool pprof heap.out
```

## 4. pprof 常用命令与分析方法

进入 pprof 交互：
```bash
go tool pprof heap.out
```

### 4.1 top —— 查看最大的内存热点
```bash
(pprof) top
```

示例输出：
```
Showing nodes accounting for 531.85MB, 96.01% of 553.94MB total
      flat  flat%   sum%        cum   cum%
  500.50MB 90.35% 90.35%   500.50MB 90.35%  bufio.NewWriterSize
   25.53MB  4.61% 94.96%    25.53MB  4.61%  some/pkg.IsTest
```

**输出字段解释**
| 字段  | 含义                             |
| ----- | -------------------------------- |
| flat  | 当前函数自己占用的内存（最关键） |
| flat% | flat / total                     |
| cum   | 当前函数 + 下游调用的总占用      |
| cum%  | cum / total                      |

**分析逻辑**
* flat 大 = 内存主要在这个函数直接分配
* cum 大、flat 小 = 内存来自其子函数

### 4.2 list —— 查看具体代码行的分配情况
```bash
(pprof) list bufio.NewWriterSize
```

**作用**：
展示函数内部 每一行代码的分配量，用于精准定位问题代码。

**示例（虚构）**：
```
Total: 500MB
     500MB   500MB     writer := &Writer{buf: make([]byte, size)}
```

### 4.3 tree —— 以调用树形式查看
```bash
(pprof) tree
```
适合查看“是谁调用了分配点”。

如果想找调用链：
```bash
(pprof) tree | grep bufio
```

### 4.4 web —— 可视化（推荐）
```bash
(pprof) web
```
会生成 graphviz 的调用图，更容易看出热点路径。

### 4.5 peek —— 关键字过滤
```bash
(pprof) peek WriterSize
```

## 5. pprof 分析内存问题的完整流程
步骤 1：抓取 heap
```bash
curl -o heap.out http://localhost:6060/debug/pprof/heap
go tool pprof heap.out
```

步骤 2：top 找热点
```bash
(pprof) top
```

如果结果像：
```
500MB bufio.NewWriterSize
```
说明是缓冲区分配过大。

步骤 3：查看调用链
```bash
(pprof) tree
```

或可视化：
```bash
(pprof) web
```
找出谁创建了这个 Writer。

步骤 4：查看代码行
```bash
(pprof) list bufio.NewWriterSize
```

步骤 5：回到代码优化

### 6. 最终总结

Go 内存排查的核心是：
* heap 看当前占用，allocs 看历史分配。
* top 定位热点，list 定位代码，tree/web 查看调用链。

通过这些信息，可以快速定位内存泄漏、超大对象分配、频繁分配等问题。

