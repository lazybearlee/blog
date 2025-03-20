---
title: Kserve-Metric-Prometheus指标
date: 2025-03-20
slug: blog-post-slug
tags:
  - Kserve
categories:
  - 笔记
description: 描述
draft: false
state: "0"
---

## **全量指标列表**

| 指标名称 | 类型 | 描述 | 关键标签 | 数值 |
|---------|-----|-----|---------|-----|
| `revision_app_request_count` | Counter | 路由到用户容器的请求数 | `configuration_name`, `container_name`, `response_code` | `1` |
| `revision_app_request_latencies` | Histogram | 响应时间（ms） | 同上 | 所有bucket均为`1`，最大值`+Inf` |
| `revision_go_alloc` | Gauge | Go语言堆分配内存（字节） | - | `4.83MB` |
| `revision_go_bucket_hash_sys` | Gauge | Go语言桶哈希系统内存（字节） | - | `1.45MB` |
| `revision_go_frees` | Gauge | Go语言对象释放次数 | - | `43166` |
| `revision_go_gc_cpu_fraction` | Gauge | GC占用CPU比例 | - | `3.21e-5` |
| `revision_go_gc_sys` | Gauge | GC系统内存（字节） | - | `2.98MB` |
| `revision_go_heap_alloc` | Gauge | Go语言堆分配内存（字节） | - | `4.83MB` |
| `revision_go_heap_idle` | Gauge | 空闲堆内存（字节） | - | `1.01MB` |
| `revision_go_heap_in_use` | Gauge | 已使用堆内存（字节） | - | `6.32MB` |
| `revision_go_heap_objects` | Gauge | 堆对象数量 | - | `15426` |
| `revision_go_heap_released` | Gauge | 释放到OS的内存（字节） | - | `745KB` |
| `revision_go_heap_sys` | Gauge | 从OS获取的堆内存（字节） | - | `7.32MB` |
| `revision_go_last_gc` | Gauge | 最后一次GC时间戳（纳秒） | - | `1.74e+18` |
| `revision_go_lookups` | Gauge | 指针查找次数 | - | `0` |
| `revision_go_mallocs` | Gauge | 对象分配次数 | - | `58592` |
| `revision_go_mcache_in_use` | Gauge | mcache结构内存（字节） | - | `28.1KB` |
| `revision_go_mcache_sys` | Gauge | mcache系统内存（字节） | - | `30.5KB` |
| `revision_go_mspan_in_use` | Gauge | mspan结构内存（字节） | - | `138KB` |
| `revision_go_mspan_sys` | Gauge | mspan系统内存（字节） | - | `143.4KB` |
| `revision_go_next_gc` | Gauge | 下次GC目标内存（字节） | - | `6.58MB` |
| `revision_go_num_forced_gc` | Gauge | 强制GC次数 | - | `0` |
| `revision_go_num_gc` | Gauge | GC总次数 | - | `5` |
| `revision_go_other_sys` | Gauge | 其他系统内存（字节） | - | `1.53MB` |
| `revision_go_stack_in_use` | Gauge | 栈内存（字节） | - | `1MB` |
| `revision_go_stack_sys` | Gauge | 栈系统内存（字节） | - | `1MB` |
| `revision_go_sys` | Gauge | 总系统内存（字节） | - | `14.5MB` |
| `revision_go_total_alloc` | Gauge | 总分配内存（字节） | - | `8.79MB` |
| `revision_go_total_gc_pause_ns` | Gauge | GC总暂停时间（纳秒） | - | `1.04e+06` |
| `revision_request_count` | Counter | 路由到队列代理的请求数 | `route_tag="DISABLED"` | `1` |
| `revision_request_latencies` | Histogram | 队列代理响应时间（ms） | 同上 | 所有bucket均为`1` |
| `process_max_fds` | Gauge | 最大文件描述符数 | - | `1.05e+06` |
| `python_info` | Gauge | Python版本信息 | - | `1`（3.11.11） |
| `process_resident_memory_bytes` | Gauge | 常驻内存（字节） | - | `266.8MB` |
| `process_start_time_seconds` | Gauge | 进程启动时间戳（秒） | - | `1.74e+09` |
| `process_cpu_seconds_total` | Counter | CPU总使用时间（秒） | - | `5.19` |
| `process_open_fds` | Gauge | 打开的文件描述符数 | - | `14` |
| `python_gc_objects_uncollectable_total` | Counter | 无法回收的GC对象数 | `generation` | 所有代均为`0` |
| `request_preprocess_seconds` | Histogram | 请求预处理时间（秒） | `model_name="sklearn-iris"` | `1`次（0.0000074秒） |
| `request_preprocess_seconds_created` | Gauge | 预处理时间指标创建时间戳 | 同上 | `1.74e+09` |
| `request_postprocess_seconds` | Histogram | 请求后处理时间（秒） | `model_name="sklearn-iris"` | `1`次（0.00000825秒） |
| `request_postprocess_seconds_created` | Gauge | 后处理时间指标创建时间戳 | 同上 | `1.74e+09` |
| `python_gc_objects_collected_total` | Counter | GC回收对象数 | `generation` | 0代`17962`，1代`6176`，2代`1170` |
| `python_gc_collections_total` | Counter | GC执行次数 | `generation` | 0代`662`，1代`60`，2代`5` |
| `process_virtual_memory_bytes` | Gauge | 虚拟内存（字节） | - | `3.9GB` |
| `request_predict_seconds` | Histogram | 模型预测时间（秒） | `model_name="sklearn-iris"` | `1`次（0.000676秒） |
| `request_predict_seconds_created` | Gauge | 预测时间指标创建时间戳 | 同上 | `1.74e+09` |