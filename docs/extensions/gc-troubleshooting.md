# 扩展阅读：GC 问题排查方法

> 本文是 [预备章：先把容量评估里的专业词讲清楚](../00-glossary-and-mental-models.md) 第 4 节的扩展内容。

## Java GC 排查

```bash
# 1. 实时看 GC 统计：每秒采样一次
$ jstat -gcutil <pid> 1000
#   S0     S1     E      O      M     CCS    YGC   YGCT  FGC  FGCT   GCT
#   0.00  45.21  67.33  32.15  95.42  91.20  1234  5.678   3   0.890  6.568
#
# 关注这几个字段：
# YGC/YGCT   — Young GC 次数和总耗时。YGCT/YGC = 平均每次暂停时间
# FGC/FGCT   — Full GC 次数和总耗时。Full GC 次数 > 0 就要警惕
# O           — Old 区使用率。持续 > 80% 说明对象晋升太快
# E           — Eden 区使用率。涨到 100% 就触发 Young GC

# 2. 开启 GC 日志（JDK 9+）
$ java -Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m -jar app.jar

# 3. 堆内存分配热点（需要先 dump）
$ jmap -dump:live,format=b,file=heap.hprof <pid>
# 然后用 MAT（Eclipse Memory Analyzer）或 JVisualVM 分析

# 4. 实时看分配热点（不暂停，推荐）
$ jcmd <pid> GC.class_stats | sort -k 3 -n -r | head -20
```

## Go GC 排查

```bash
# 1. 启动时开启 GC trace
$ GODEBUG=gctrace=1 ./your-app
# gc 1 @0.012s 2%: 0.026+1.2+0.019 ms clock, 0.21+0.34/1.0/0.42+0.15 ms cpu, 4->6->2 MB, 5 MB goal, 0 MB stacks, 0 MB globals, 8 P
# 关注：
# gc N      — 第几次 GC
# 2%        — GC 占总 CPU 的比例（> 5% 需要关注）
# 4->6->2 MB — GC 前堆大小->GC 中峰值->GC 后存活量
# 8 P       — 并行度

# 2. 通过 HTTP pprof 看内存分配热点
$ go tool pprof http://localhost:6060/debug/pprof/heap
# 或者看累积分配量（找分配热点更有效）
$ go tool pprof http://localhost:6060/debug/pprof/allocs

# 3. 在 pprof 交互模式下
$ (pprof) top 20       # 看分配最多的函数
$ (pprof) web           # 生成调用图（需要安装 graphviz）
```

## 通用排查思路

```text
第一步：确认是不是 GC 问题
  → 看 GC 日志/统计，暂停时间是否异常，频率是否过高

第二步：确认 GC 为什么频繁
  → 看堆内存分配速率（每秒分配多少）
  → 看 GC 后存活对象量（回收不掉说明有引用泄漏）

第三步：定位分配热点
  → dump 堆内存分析大对象
  → pprof 分析分配调用栈

第四步：对症下药
  → 对象池/缓存复用（减少分配）
  → 减少不必要的大对象创建
  → 调整 GC 参数（最后手段）
```

## 四种典型 GC 症状及解法

### 症状一：P99 周期性飙高

延迟平时很好，每隔几秒或几十秒就出现一次尖刺。

```text
典型原因：Young GC 暂停时间过长
排查方法：
  1. jstat -gcutil 看 YGC 次数，计算 YGCT/YGC = 平均暂停时间
  2. 如果每次暂停 > 50ms，看 Young 区是否太大
  3. 如果暂停时间合理但频率太高，说明分配速率过高
解决思路：
  - 减少短生命周期对象创建（对象池、StringBuilder 替代 String 拼接）
  - 减少 Young 区大小以降低单次暂停时间（但会增加 GC 频率，需权衡）
  - 升级到 ZGC 或 Shenandoah（如果 Java 版本允许）
```

### 症状二：Full GC 频繁出现

Old 区使用率持续上涨，Full GC 每隔几分钟就来一次，每次暂停几百毫秒甚至几秒。

```text
典型原因：内存泄漏或老年代过早填满
排查方法：
  1. jstat -gcutil 看 O 列是否持续 > 80%
  2. FGC 次数是否持续增加
  3. jmap -dump:live 导出堆，用 MAT 分析 Retained Heap 最大的对象
解决思路：
  - 找到泄漏的对象（没被释放的集合、缓存没有淘汰策略、监听器没注销）
  - 增大堆内存（治标不治本）
  - 检查是否有大对象直接分配到 Old 区
```

### 症状三：GC 占 CPU 过高

系统 CPU 使用率高，但业务吞吐量没增加。

```text
典型原因：分配速率过高，GC 线程忙不过来
排查方法：
  1. top 看 CPU 使用率，如果 User CPU 高但业务 QPS 没涨，可能 CPU 花在 GC 上
  2. Go：GODEBUG=gctrace=1 看 GC 占 CPU 百分比（正常 < 5%）
解决思路：
  - 用 pprof 或 jmap 定位分配热点函数
  - 减少 JSON 序列化/反序列化时的临时对象
  - 复用 buffer 和对象池
```

### 症状四：Go 服务内存持续上涨但不 OOM

RSS 长期上涨，但 GC 后存活量并不大。

```text
典型原因：Go 默认不会立刻把释放的内存归还操作系统
排查方法：
  1. 对比 GC 后存活量和 RSS，如果差距很大，说明内存被 Go 运行时缓存了
  2. 检查 GOMEMLIMIT 是否设置
解决思路：
  - Go 1.19+ 设置 GOMEMLIMIT 让 GC 更主动地回收
  - 设置 GOGC 降低 GC 触发阈值（默认 100，设为 50 会更频繁 GC 但内存更少）
  - 极端情况调用 debug.FreeOSMemory() 强制归还（仅用于排查）
```

## 生产环境排查的安全注意事项

```text
jmap -dump:live  会触发 Full GC，会在暂停期间产生大 IO，高峰期慎用
kill -3 <pid>    向 Java 进程发送 SIGQUIT 会打印线程 dump，但不暂停业务
strace -p <pid>  会严重拖慢进程，只在隔离环境中使用
perf record      对性能影响小（< 3%），可以在生产环境安全使用
```
