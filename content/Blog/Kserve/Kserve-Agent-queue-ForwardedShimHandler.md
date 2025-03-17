---
title: Kserve-Agent-queue-ForwardedShimHandler
date: 2025-03-17
slug: blog-post-slug
tags:
  - Serverless
  - Knative
  - Kserve
categories:
  - Blog
description: 描述
draft: false
state: "0"
---
在kserve agent中用到了如下的knative库：
```go
// cmd/agent/main.go
composedHandler = queue.ForwardedShimHandler(composedHandler)
```

---

在 Knative 的 HTTP 流量管理中，ForwardedShimHandler 用于将传统的 X-Forwarded-\* 头转换为符合 RFC7239 标准的 Forwarded 头。其主要动机包括：

1. **标准化请求头**  
   RFC7239 定义了 Forwarded 头的格式，提供更结构化的信息（如 for、host、proto 等）。而许多环境中，代理层会设置 X-Forwarded-For、X-Forwarded-Proto 和 X-Forwarded-Host 等头。ForwardedShimHandler 的作用就是利用这些头的信息构造一个标准的 Forwarded 头，方便后续组件统一解析。

2. **兼容性和安全性**  
   如果请求中已经存在 Forwarded 头（可能由上游代理设置），Handler 不做任何处理，从而避免重复或冲突。同时，仅组合可信的 X-Forwarded-\* 信息。

---

### 代码

下面是 ForwardedShimHandler 的核心代码片段：

````go
func ForwardedShimHandler(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer h.ServeHTTP(w, r)

		// 如果已有 Forwarded 头，则不做处理
		fwd := r.Header.Get("Forwarded")
		if fwd != "" {
			return
		}

		// 获取传统的 X-Forwarded-\* 头信息
		xff := r.Header.Get("X-Forwarded-For")
		xfp := r.Header.Get("X-Forwarded-Proto")
		xfh := r.Header.Get("X-Forwarded-Host")

		// 没有这些头信息则直接返回
		if xff == "" && xfp == "" && xfh == "" {
			return
		}

		// 根据获取的 X-Forwarded-\* 头信息生成 Forwarded 头
		r.Header.Set("Forwarded", generateForwarded(xff, xfp, xfh))
	})
}
````

分析过程：

- **检查和保护已有的 Forwarded 头**  
  一开始通过 `r.Header.Get("Forwarded")` 判断请求中是否已经存在标准 Forwarded 头。如存在，直接返回，不再更改，确保不覆盖已存在的值。

- **读取 X-Forwarded-\* 头**  
  分别获取 X-Forwarded-For、X-Forwarded-Proto 和 X-Forwarded-Host 头。如果这些头都没有设置，则说明没有相关信息可用，Handler 也不会做额外的处理。

- **构造标准 Forwarded 头**  
  调用 generateForwarded 函数，将上面读取到的信息整合成标准格式，如：
  
  ```
  Forwarded: for=<client>;host=<host>;proto=<protocol>
  ```
  
  在生成过程中还考虑了多个代理节点的情况，即 X-Forwarded-For 可能包含多个地址，通过逗号分隔，对每个节点均构造 "for" 域。

---

### 应用场景

在 Knative 的代理和负载均衡中，通过 ForwardedShimHandler 可以：
  
- 为下游组件提供标准化后的请求头，从而简化对请求来源、主机和协议的处理；
- 消除不同代理实现之间因头信息不一致带来的问题，提高请求转发的兼容性；
- 辅助日志、监控等功能更容易解析和展示客户端信息。

---

总体来看，ForwardedShimHandler 是 Knative 为兼容老式 X-Forwarded-\* 头而新增的一个“shim”层，确保即便上游仅提供非标准头信息，也能生成正确的 Forwarded 头供后续使用。
