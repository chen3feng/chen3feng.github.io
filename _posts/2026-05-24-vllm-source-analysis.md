---
title: "我用 AI 读完了 vLLM 源码，写了一份深度分析笔记"
date: "2026-05-24 20:00:00 +0800"
categories: [LLM, System]
tags: [vLLM, PagedAttention, FlashAttention, LLM Inference, Source Code]
layout: post
published: true
---

最近花了些时间，用 AI 辅助系统性地阅读了 [vLLM](https://github.com/vllm-project/vllm) 的源码，整理成了一份深度分析文档集：

👉 **[vllm-analysis](https://github.com/chen3feng/vllm-analysis)**

## 为什么读 vLLM 源码

我是一名资深后台开发工程师，过去十几年一直在做服务端性能优化和系统架构——CPU 缓存行对齐、内存池设计、无锁队列这些是日常。进入 AI 时代，技能栈需要升级，但回头一看，LLM 推理引擎这个方向其实最对味：它本质上是**显存管理 + 算子调度 + 分布式通信**的系统工程，和我之前的积累高度重合。

于是我选了 vLLM 作为切入点。它是当前最流行的 LLM 推理引擎，以 **PagedAttention** 技术闻名——借鉴操作系统虚拟内存的思想，将 KV cache 划分为固定大小的 block（页），一举消除了传统引擎因预分配连续显存导致的严重碎片问题。官方数据显示内存浪费不到 4%，吞吐量相比 HuggingFace Transformers 最高提升 **24 倍**。

在开源之前，vLLM 已支撑了 Chatbot Arena 数百万用户的 Vicuna 对话服务——内部基准测试显示吞吐提升高达 **30 倍**。此后被大量公司集成到生产系统，社区贡献者超过 2000 人。

但 vLLM 源码规模庞大（Python 约 20 万行 + C++/CUDA 数万行 + Rust 前端），架构复杂（多进程通信、三套算子注册机制、六种 attention backend、数十种量化方法）。官方文档侧重于使用和配置，对内部实现鲜有涉及。这也是我想深入源码的原因。

## 笔记涵盖什么

8 篇文章，由浅入深：

| # | 主题 |
|---|------|
| 1 | **架构概述** — 六层架构、请求的完整生命周期、关键子系统 |
| 2 | **代码结构分析** — 目录组织、CMake/setuptools 构建系统、C++/CUDA/Rust 代码分工 |
| 3 | **初始化流程分析** — 从 `LLM()` 调用到 Engine 就绪：配置解析、Worker 创建、显存 Profiling、KV cache 分配 |
| 4 | **模型加载流程分析** — 9 种 ModelLoader、600+ 模型注册表、TP 分片权重加载 |
| 5 | **推理流程分析** — 一个 step 的完整路径：调度 → Attention → 采样 → 输出 |
| 6 | **算子注册与分发** — `torch.ops._C.*` vs `torch.ops.vllm.*` vs `@triton.jit`，三种算子注册机制的完整剖析 |
| 7 | **FlashAttention 实现分析** — vLLM 的默认 attention 后端：FA2/FA3/FA4 版本选择、cascade attention、DCP |
| 8 | **PagedAttention 实现分析** — Block Table 如何工作（纯软件查表，不依赖 GPU MMU）、KV cache 内存管理、CUDA kernel 逐段解析、传统引擎为何没有这种机制 |

## 特点

- 所有代码引用都是**可点击链接**，精确到源码行号
- vLLM 源码通过 git submodule 固定在 `5bb8d2767`，行号永久有效
- 由浅入深——从架构全景逐步深入到 CUDA kernel 的地址翻译细节
- 在线版：[https://blog.chen3feng.top/vllm-analysis/](https://blog.chen3feng.top/vllm-analysis/)（搜索 + 侧边栏导航）

## 生成方法

这些笔记由 **Claude Code** 通过系统性阅读 vLLM 源码生成。流程是：

1. **确定分析角度** — 每篇围绕一个核心问题（"PagedAttention 如何实现？""一次推理经过哪些步骤？"）
2. **逐层探索** — 从入口点跟踪调用链，找到关键类和函数后读取具体实现，行号全部精确查找
3. **生成结构化文档** — ASCII 架构图、调用链、可点击代码引用、交叉引用
4. **事实核查** — 最近一轮对 300+ 个行号引用做逐行验证，修正了 10 个严重错误

篇幅和深度远非一篇博客能容纳，感兴趣的朋友可以直接翻阅文档集。欢迎指正和补充。
