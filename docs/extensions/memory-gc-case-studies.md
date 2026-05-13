# 扩展阅读：生产环境内存与 GC 问题案例集

> 本文是 [预备章：先把容量评估里的专业词讲清楚](../00-glossary-and-mental-models.md) 第 4 节的扩展内容。

## 案例一：JSON 序列化引发的 GC 风暴

```text
现象：一个 API 网关，平时 P99 延迟 20ms，流量翻倍后 P99 飙到 500ms。
排查过程：
  1. jstat 看到 YGC 从每秒 2 次涨到每秒 15 次
  2. GC 日志显示每次 Young GC 暂停 30ms，加上频率高，每秒有 450ms 在做 GC
  3. pprof 定位到分配热点是 JSON 序列化——每处理一个请求，Gson 创建大量临时 Token 对象
根因：
  每个请求序列化一个大 JSON 响应体（包含 500 个商品详情），Gson 每次创建 2000+ 个临时对象。
  流量翻倍后对象分配速率达 400MB/s，GC 完全跟不上。
解决方案：
  1. 换用 Jackson 流式序列化，对象分配量降低 80%
  2. 对响应体做缓存——相同参数的请求在 TTL 内直接返回缓存结果
  3. 最终 P99 回到 30ms，GC 频率降到每秒 3 次
```

## 案例二：缓存无限增长导致 OOM

```text
现象：一个推荐服务，启动后运行稳定，但 3-5 天后必定 OOM 重启。
排查过程：
  1. jstat 看到 O 列（Old 区使用率）从启动时的 20% 缓慢涨到 95%
  2. 每次 Full GC 回收不掉多少对象——大量对象被永久持有
  3. MAT 分析 heap dump，发现一个 ConcurrentHashMap 存了 300 万个用户推荐结果，每个 2KB，总计 6GB
根因：
  开发者为"提高性能"把推荐结果全部缓存在本地 Map 里，没有过期策略，没有大小限制。
  用户量增长后，缓存越来越大，最终吃光堆内存。
解决方案：
  1. 用 Caffeine/Guava Cache 替代裸 HashMap，设置最大条目数和过期时间
  2. 设置缓存淘汰策略（LRU 或 W-TinyLFU）
  3. 内存占用稳定在 2GB 以内，不再 OOM
```

## 案例三：日志框架引发的 Off-Heap 泄漏

```text
现象：一个 Java 服务 RSS 持续增长，但 jstat 显示 Heap 使用率稳定在 50%。
排查过程：
  1. cat /proc/<pid>/smaps_rollup 发现大量匿名内存（Anonymous）
  2. pmap -x <pid> 发现多段 64MB 的匿名映射
  3. 排查发现 Log4j2 的 AsyncAppender 内部 RingBuffer 队列设得过大（100 万个槽位），
     每个日志事件占 4KB，队列满时就是 4GB Off-Heap 内存
根因：
  日志队列配置不合理。日志写入速度跟不上产生速度时，队列不断扩容，内存持续增长。
  因为是 Off-Heap，GC 管不到。
解决方案：
  1. 减小 RingBuffer 大小为 4096（默认值）
  2. 设置 DiscardThreshold，队列满时丢弃低级别日志（DEBUG/INFO），保留 ERROR
  3. RSS 稳定在 1.5GB 左右
```

## 案例四：Go 服务延迟毛刺

```text
现象：Go 微服务 P99 每隔 1-2 分钟出现一次 100ms 的延迟毛刺，P50 始终正常。
排查过程：
  1. GODEBUG=gctrace=1 看到 GC 间隔约 60 秒，每次 STW 暂停约 0.3ms
     → Go 的 GC 暂停不是原因
  2. 仔细看 gctrace 输出：gc 后堆 400MB，但 RSS 是 2GB
     → Go 运行时持有大量缓存内存
  3. 进一步发现，每次 GC 触发前后有短暂的 CPU 飙升（标记阶段占 15% CPU）
     → GC 标记阶段的 CPU 竞争导致业务 goroutine 被调度延迟
根因：
  服务在处理请求时大量使用 bytes.Buffer 拼接字符串，分配速率达 200MB/s。
  GC 标记阶段 CPU 开销高，在高流量时段与业务线程争抢 CPU。
解决方案：
  1. 引入 sync.Pool 复用 bytes.Buffer，分配速率降到 20MB/s
  2. 设置 GOMEMLIMIT=1536MB 让 GC 更积极回收，避免 RSS 无限膨胀
  3. GC 频率不变但 CPU 开销降低到 3%，P99 毛刺消失
```

## 案例五：连接池与线程池叠加导致的内存爆炸

```text
现象：一个电商服务大促时多个实例同时 OOM，重启后几分钟内再次 OOM。
排查过程：
  1. dmesg 看到 OOM Killer 杀掉了 Java 进程
  2. jstat 看到 Heap 满了，Full GC 间隔从正常的 10 分钟缩短到 30 秒
  3. MAT 分析发现：
     - 200 个 HTTP 连接 × 每个连接 2MB buffer = 400MB
     - 200 个数据库连接 × 每个连接的 ResultSet 缓存 5MB = 1GB
     - 1000 个线程 × 每个线程栈 1MB = 1GB
     总计 2.4GB，而堆只配了 2GB
根因：
  大促期间运维把线程池从 200 调到 1000"以防万一"，数据库连接池也从 50 调到 200。
  但每个连接和线程都要占内存，叠加后超出堆容量。
解决方案：
  1. 线程池恢复为 CPU 核心数 × 2，用异步处理来应对并发
  2. 数据库连接池设为核心数的 2-3 倍（连接数不是越多越好）
  3. 堆内存从 2GB 调到 4GB（容器规格同步升级）
  4. 监控加上 Heap 使用率告警（> 80% 持续 5 分钟触发）
```

## 内存问题排查实用技巧汇总

### 快速定位阶段（5 分钟内）

```bash
# 1. 系统级：free -h 看 available 和 swap
# 2. 进程级：top 按内存排序（Shift+M），找 RSS 最大的进程
# 3. 趋势：pidstat -r 10 -p <pid>，看 RSS 是涨是稳
# 4. GC：jstat -gcutil <pid> 1000（Java）或 GODEBUG=gctrace=1（Go）
```

### 深入分析阶段（15-30 分钟）

```bash
# 1. 堆 dump（Java，会暂停几十毫秒到几秒）
$ jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>

# 2. Go heap profile（不暂停）
$ curl http://localhost:6060/debug/pprof/heap > /tmp/heap.pb.gz
$ go tool pprof -http=:8080 /tmp/heap.pb.gz

# 3. 看 RSS 的详细组成
$ cat /proc/<pid>/smaps_rollup
# 关注 Anonymous（堆+栈）、Mapped（mmap 文件）、Shared（共享内存）

# 4. 检查是否有 FD 泄漏导致的内存增长
$ ls /proc/<pid>/fd | wc -l
# FD 数量持续增长通常意味着连接或文件泄漏
```

### 排查顺序口诀

```text
free 看全局 → top 找进程 → pidstat 看趋势
→ jstat/pprof 看 GC → dump 堆分析对象 → 定位代码修复
```
