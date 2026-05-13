# 容量评估：架构师必须补上的一门工程课

所谓容量评估，本质上不是问“系统能扛多少 QPS”，而是问一个更根本的问题：这个系统在未来某个规模下，是否还能以可接受的成本、可接受的稳定性、可接受的用户体验继续运行。

这份内容已经从单篇长文拆分为独立章节。每一章都补充了更完整的背景、实现原理、生产实践、常见坑和可操作清单，适合初学者按顺序阅读，也适合工程团队作为容量评估手册使用。

## 目录

| 章节 | 文档 | 重点 |
| --- | --- | --- |
| 预备章 | [先把容量评估里的专业词讲清楚](docs/00-glossary-and-mental-models.md) | QPS、P99、CPU Load、Buffer Pool、热 Key、Kafka Lag、KV Cache 等术语的白话解释 |
| 第一章 | [什么是容量评估](docs/01-what-is-capacity-planning.md) | 容量评估的定义、目标、最小可用模型和基础检查清单 |
| 第二章 | [为什么很多工程师不懂容量评估](docs/02-why-engineers-miss-capacity.md) | 常见误解、规模效应、QPS 误区、平均值误区和团队分工 |
| 第三章 | [容量评估到底评估什么](docs/03-what-to-evaluate.md) | 流量、存储、计算、内存、网络、数据库、Redis、Kafka 和 AI 推理 |
| 第四章 | [真正高级的容量评估能力](docs/04-advanced-capacity-methods.md) | 用户行为模型、瓶颈定位、压测体系、成本模型和 SLO |
| 第五章 | [不同阶段的工程师应该达到什么水平](docs/05-engineer-growth-stages.md) | 初级、中级、高级、架构师和 SRE 的能力要求 |
| 第六章 | [推荐学习路线](docs/06-learning-path.md) | Linux、网络、数据库、性能分析、压测、分布式系统和 AI 容量 |
| 第七章 | [以 Typeflux 为案例：实时 AI 产品最该先学什么](docs/07-typeflux-priorities.md) | AI 推理、WebSocket、Redis/Kafka 和成本优化 |
| 第八章 | [最有效的实践方式](docs/08-practical-projects.md) | API 压测、Redis 实验、WebSocket 实验和 AI 推理实验 |
| 第九章 | [最终要形成的思维方式](docs/09-capacity-mindset.md) | 从功能到系统、从平均值到分布、从救火到机制 |
| 附录 | [容量评估模板与落地清单](docs/10-capacity-planning-template.md) | 可直接复用的容量评估、压测、降级、成本和复盘模板 |
| 实战案例 | [以 Typeflux 为案例：从零做一次容量评估](docs/11-end-to-end-case-study.md) | 从 DAU 推导长连接、带宽、ASR、LLM、Redis、MySQL、成本和压测计划 |

## 阅读建议

如果你是初学者，建议先读预备章，把专业词弄清楚，再从第一章读到第六章，建立基本概念和学习路线，最后做第八章的实验。

如果你已经负责线上系统，可以优先阅读第三章、第四章和附录，把资源评估、压测报告和容量评审清单用到当前项目里。

Typeflux 是作者的开源项目，在这里作为贯穿案例使用。读者不需要正在开发 Typeflux，也可以通过第七章和实战案例理解实时 AI 产品的容量评估方法。

部分章节包含深度扩展内容，放在 [docs/extensions/](docs/extensions/) 目录下。主文档保持精炼，扩展文档适合想要深入某个方向的读者按需阅读。

## 核心方法

整套内容围绕一条链路展开：

```text
用户规模
→ 用户行为模型
→ 请求链路
→ 资源消耗
→ 系统瓶颈
→ 扩容方案
→ 成本模型
→ 风险模型
```

容量评估不是一门单纯的运维课，而是一门工程判断课。它训练的是工程师对系统边界、增长规律和资源约束的理解。
