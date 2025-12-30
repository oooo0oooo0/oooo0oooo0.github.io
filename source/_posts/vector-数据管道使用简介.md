---
title: vector 数据管道使用简介
date: 2025-12-30 15:05:15
tags:
---

# Vector 在 Syslog 采集与数据融合体系中的实践与踩坑总结

> 本文分享一次在生产环境中使用 Vector 作为 Syslog 统一接入层的实践经验，涵盖
> * TCP / UDP Syslog 接入
> * Docker 部署注意事项
> * 文件落盘作为缓冲
> * 多业务动态输出
> * 与后续数据同步、ES、融合任务的整体架构设计

## 一、整体架构设计

在整个系统中，Vector 的定位非常清晰：

> **只负责“稳定接入 + 标准化 + 落盘”，不承载任何业务逻辑**

整体链路如下：

``` mermaid
flowchart LR
    %% ======================
    %% 数据源
    %% ======================
    subgraph Sources["Syslog / 设备 外部系统"]
        D1[认证日志<br>syslog形式]
    end

    %% ======================
    %% Vector 容器
    %% ======================
    subgraph VectorContainer["adapter container"]
        V[Vector服务<br>Syslog Source]
        F[文件Sink<br>1小时滚动]
    end

    %% ======================
    %% 共享存储
    %% ======================
    subgraph SharedVolume["共享 Volume"]
        S[/Syslog 文件目录/]
    end

    %% ======================
    %% 同步 容器
    %% ======================
    subgraph SyncContainer["sync container"]
        Sync[同步服务<br>解析文件]
    end

    %% ======================
    %% 后端服务
    %% ======================
    subgraph Backend["后端基础设施"]
        ES[(Elasticsearch)]
        MQ[(Redis<br>消息队列)]
    end

    %% ======================
    %% 融合服务
    %% ======================
    subgraph Fusion["merge container"]
        Consumer[融合任务消费者]
        Calc[融合计算逻辑]
    end

    %% ======================
    %% 数据流
    %% ======================
    D1 -->|tcp/udp主动推送| V

    V --> F
    F --> S

    S --> Sync
    Sync -->|写入过程数据| ES
    Sync -->|下发融合任务| MQ

    MQ --> Consumer
    Consumer --> Calc
    Calc -->|读取原始/过程数据| ES
    Calc -->|写回融合结果| ES
```

**设计关键点**

* Vector 独立容器，职责单一
* 文件作为天然缓冲层（削峰、回放、解耦）
* 同步服务与融合服务完全异步
* ES 作为统一事实源（process + result）

## 二、Vector Syslog 接入配置（TCP + UDP）
### 1. 为什么同时支持 TCP 和 UDP

* UDP
  * 设备、网络产品默认协议
  * 简单、高性能
  * 可接受少量丢失

* TCP
  * 应用日志、关键系统
  * 保证送达
  * 易于调试

因此 Vector 同时监听两个端口。

### 2. Syslog Source 配置（RFC3164）

```yaml
sources:
  syslog_in:
    type: syslog
    mode: tcp
    address: "0.0.0.0:9000"
    protocol: rfc3164
    framing: non-transparent   # 允许使用换行分隔

  syslog_in_udp:
    type: syslog
    mode: udp
    address: "0.0.0.0:514"
    protocol: rfc3164
```

**关键说明**

* protocol: rfc3164
  * 兼容性最好（设备/系统支持度最高）,否则默认使用 RFC5424，老旧设备可能不支持

* TCP 下显式设置：
    `framing: non-transparent`

## 三、Docker 部署的一个“致命细节”
### UDP 端口映射必须显式声明

❌ 错误示例（可能导致 UDP 收不到）：
``` bash
-p 514:514
```

✅ 正确做法：
``` bash
docker run -d --name vector \
  -p 9000:9000/tcp \
  -p 514:514/udp \
  -v /data/vector:/etc/vector \
  timberio/vector:nightly-alpine \
  -c /etc/vector/vector-syslog.yaml
```

> **结论**：TCP 正常、UDP 收不到，第一时间检查 Docker 端口协议映射。

## 四、文件落盘：Vector 最被低估的能力
### 为什么不用直接推 ES？
| 方案              | 问题                                                             |
| ----------------- | ---------------------------------------------------------------- |
| Vector → ES       | 强耦合、ES 抖动风险, 主业务稳定性要求（防止日志流量波动压垮 ES） |
| Vector → MQ       | 顺序/重放复杂，引入额外组件                                      |
| **Vector → 文件** | ✅ 解耦 / 缓冲 / 可回放 / 成本可控                                |

### 文件 Sink 配置（按小时滚动）

```yaml
sinks:
  syslog_file:
    type: file
    inputs:
      - syslog_in
      - syslog_in_udp
    path: /var/vector/data/vector-%Y-%m-%d-%H.log
    encoding:
      codec: json
    buffer:
      type: memory
      max_size: 10MB
```

> Vector 的 memory buffer 的 max_events 默认单位是事件数，而 max_size 应该是字节数（如 104857600 或 100MB）。建议根据实际日志大小调整此值。

### 文件清理策略

Vector 本身不提供文件清理功能，需要配合外部脚本实现。以下是两种常用策略：

#### 策略一：按时间清理（删除 5 天前的文件）

```bash
#!/bin/bash
# cleanup_by_time.sh
# 删除 5 天前的日志文件

LOG_DIR="/var/vector/data"
RETENTION_DAYS=5

find "$LOG_DIR" -name "*.log" -type f -mtime +$RETENTION_DAYS -delete

echo "$(date '+%Y-%m-%d %H:%M:%S') 已清理 ${RETENTION_DAYS} 天前的日志文件"
```

配合 crontab 定时执行：
```bash
# 每天凌晨 3 点执行清理
0 3 * * * /opt/scripts/cleanup_by_time.sh >> /var/log/vector_cleanup.log 2>&1
```

#### 策略二：按目录大小清理（超过 1GB 删除最早文件）

```bash
#!/bin/bash
# cleanup_by_size.sh
# 当目录超过 1GB 时，删除最早的文件直到低于阈值

LOG_DIR="/var/vector/data"
MAX_SIZE_MB=1024  # 1GB

get_dir_size_mb() {
    du -sm "$LOG_DIR" 2>/dev/null | cut -f1
}

current_size=$(get_dir_size_mb)

while [ "$current_size" -gt "$MAX_SIZE_MB" ]; do
    # 找到最早的文件并删除
    oldest_file=$(find "$LOG_DIR" -name "*.log" -type f -printf '%T+ %p\n' 2>/dev/null | sort | head -1 | cut -d' ' -f2-)

    if [ -z "$oldest_file" ]; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') 没有找到可删除的文件"
        break
    fi

    rm -f "$oldest_file"
    echo "$(date '+%Y-%m-%d %H:%M:%S') 已删除: $oldest_file"

    current_size=$(get_dir_size_mb)
done

echo "$(date '+%Y-%m-%d %H:%M:%S') 当前目录大小: ${current_size}MB"
```

配合 crontab 定时执行：
```bash
# 每小时检查一次目录大小
0 * * * * /opt/scripts/cleanup_by_size.sh >> /var/log/vector_cleanup.log 2>&1
```

#### 两种策略对比

| 策略   | 优点                     | 缺点                   | 适用场景           |
| ------ | ------------------------ | ---------------------- | ------------------ |
| 按时间 | 简单可预测，保证回溯窗口 | 磁盘占用不可控         | 日志量稳定的环境   |
| 按大小 | 磁盘占用可控             | 高峰期可能丢失较新数据 | 磁盘空间有限的环境 |

> **生产建议**：可以同时启用两种策略，按时间清理作为主策略，按大小清理作为兜底保护。

## 五、多业务动态输出（无需动态 sink）

Vector **不支持运行时动态新增 sink**，但支持更优雅的方式。

### 推荐做法：path 模板 + 事件字段

```yaml
path: /var/vector/data/{{ biz }}/%Y-%m-%d-%H.log
```
配合 `remap`：
``` yaml
transforms:
  normalize:
    type: remap
    inputs: [syslog_in, syslog_in_udp]
    source: |
      if .appname == "nginx" {
        .biz = "web"
      } else if .appname == "sshd" {
        .biz = "security"
      } else {
        .biz = "default"
      }
```

### 带来的好处
* 新业务 无需修改 Vector 配置
* 目录天然成为业务边界
* 同步逻辑保持通用

## 六、调试利器：Console Sink

接入新设备时，经常面临一个问题：

> 到底有没有日志发过来？

解决方案：加一个控制台输出。

```yaml
sinks:
  syslog_console:
    type: console
    inputs:
      - syslog_in
      - syslog_in_udp
    encoding:
      codec: json
```

## 七、Syslog 测试与排坑总结
### 1. logger 命令的隐藏坑
``` bash
logger --tcp   # 强制 TCP
logger --udp   # 强制 UDP
# 或简写
logger -T   # TCP(部分版本)
logger -d   # UDP
```
❌ 错误示例（以为是 UDP，实际是 TCP）：
``` bash
logger -n 1.2.3.4 -P 514 -T "msg"
```
✅ 正确UDP：
``` bash
logger -n 1.2.3.4 -P 514 -d "msg"
```

### 2. TCP framing 报错的本质
``` bash
Failed framing bytes
Unable to decode message len as number
```
原因：
* Vector 默认 TCP 使用 octet-counting
* 客户端没发长度
解决：
``` yaml
framing: non-transparent
``` 

## 八、总结：Vector 在这套体系中的价值

> **Vector ≠ 日志分析工具**
> 
> **Vector = 高性能、可控、可组合的日志“入口层”**

### 它适合做什么

* Syslog / 文件 / TCP / UDP 统一接入
* 标准化、结构化
* 稳定落盘

### 它不该做什么

* 业务逻辑
* 融合计算
* ES 强绑定

### 一些实践建议

* 生产环境 优先 RFC3164
* TCP 明确 framing，UDP 明确端口映射
* 文件作为“第一落点”
* 业务扩展靠字段和目录，不靠配置膨胀