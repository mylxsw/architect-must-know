# 扩展阅读：文件描述符（FD）深度解析

> 本文是 [预备章：先把容量评估里的专业词讲清楚](../00-glossary-and-mental-models.md) 第 3 节的扩展内容。

## 什么是文件描述符

Linux 有一个核心设计哲学：**一切皆文件（Everything is a file）** 。不只是磁盘上的文件，网络连接、管道、设备，在内核看来都是"文件"，都需要一个文件描述符（File Descriptor，简称 FD）来标识。

| 对象 | 是否占用 FD |
| --- | --- |
| 普通文件 | ✅ |
| TCP 连接 | ✅ |
| WebSocket 连接 | ✅ |
| Unix Socket | ✅ |
| Pipe 管道 | ✅ |
| epoll 实例 | ✅ |
| stdin/stdout/stderr | ✅ |

一个 WebSocket 服务，每个客户端连接至少占用 1 个 FD。如果默认上限是 1024，扣掉系统自身占用的几十个，实际只能接受几百个连接。

## 为什么历史默认值只有 1024

**第一，早期系统并发很低。** 早期 Unix 和 Linux 的时代，没有 WebSocket、长连接、微服务这些东西。当时一台机器同时打开 1024 个文件已经算很多了。

**第二，FD 不只是一个数字，它背后是内核资源。** 每一个 FD 背后都有内核对象：socket buffer、inode、file struct、dentry、page cache、TCP 状态机。如果不加限制，10 万个连接可能直接吃掉几个 GB 内存。

**第三，防止程序 Bug 把系统打死。** 一个经典 Bug 是 FD 泄漏：程序不断打开文件或连接，却忘记关闭。1024 这个默认值本质上是一个安全保护网。

**第四，早期 select() 系统调用的硬限制。** `select()` 内部使用固定大小的位图（bitmap），编译时就确定了上限：

```text
FD_SETSIZE = 1024
```

后来才出现了更好的替代方案：

| 技术 | 改进 | 适用系统 |
| --- | --- | --- |
| poll | 不再有 1024 硬限制，但仍是 O(n) 遍历 | Linux/Unix |
| epoll | O(1) 事件通知，适合高并发 | Linux |
| kqueue | 类似 epoll 的高效方案 | macOS/BSD |
| io_uring | 新一代异步 IO，更高性能 | Linux 5.1+ |

> 初学者提示：如果你用 Go、Java、Node.js 等高级语言写网络服务，底层的 epoll/kqueue 通常由运行时或框架自动处理，你不需要直接操作它们。但理解它们的存在和原理，有助于你理解为什么现代服务器能处理几十万并发连接。

**第五，兼容低配设备。** Linux 必须能运行在各种硬件上：小 VPS、嵌入式设备、树莓派。如果默认就开放 100 万 FD，很多低配机器启动时就会预分配大量内核资源。

## 现代系统为什么需要调大

现代系统大量使用长连接，FD 消耗远超早期设计预期：

| 技术 | FD 消耗特点 |
| --- | --- |
| WebSocket | 每个客户端一个长连接，持续占用 |
| gRPC | 多路复用但仍需连接 |
| Redis 连接池 | 每个应用实例维持多个连接 |
| Kafka | Broker 维护 client、replica、controller 多种连接 |
| 微服务 | 服务间大量 HTTP/gRPC 连接 |
| AI 流式推理 | 长时间占用连接直到生成完成 |

## 怎么调整 FD 上限

Linux 的 FD 限制有多层，必须逐层检查：

| 层级 | 查看方式 | 修改方式 |
| --- | --- | --- |
| 进程级（soft/hard limit） | `ulimit -n` | `/etc/security/limits.conf` 或 `ulimit -n 65535` |
| 系统级 | `cat /proc/sys/fs/file-max` | `sysctl fs.file-max=2000000` |
| systemd 服务级 | 服务配置文件 | `LimitNOFILE=65535` |
| 容器级（Docker/K8s） | `docker inspect` 或 Pod spec | `--ulimit nofile=65535:65535` |

常见的推荐值：

| 场景 | 建议 FD 上限 |
| --- | --- |
| 普通 Web 服务 | 65535 |
| 高并发网关（Nginx、Envoy） | 500000+ |
| 超大规模长连接服务 | 1000000+ |

## 调大 FD 不是终点

真正高级的工程师不会只改一个 `ulimit` 就觉得万事大吉。还要关注：

1. **FD 泄漏**。程序打开连接或文件后忘记关闭，FD 数量持续增长。Go 里忘写 `defer file.Close()`、Java 里忘关 InputStream，都是常见原因。
2. **TIME_WAIT 堆积**。大量短连接关闭后进入 TIME_WAIT 状态，虽然不再活跃，但仍占用 FD 和端口。
3. **epoll 是否正确使用**。如果事件循环写得有问题，FD 注册了却没有正确移除，会导致 FD 泄漏和 CPU 飙升。
4. **单机连接模型要整体评估**。10 万个 WebSocket 连接不只是 10 万个 FD，还意味着 10 万份 TCP buffer、10 万个内核 socket 对象、对应的 epoll 事件、应用层的 goroutine 或线程。

可以这样记：

```text
FD 上限决定你"能不能"打开这么多连接
内存和 CPU 决定你"撑不撑得住"这么多连接
```
