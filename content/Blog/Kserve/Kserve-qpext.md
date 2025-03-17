---
title: Kserve-qpext-queue proxy扩展
date: 2025-03-17
slug: blog-post-slug
tags:
  - Kserve
  - Knative
  - Serverless
categories:
  - Blog
description: 描述
draft: false
state: "0"
---


---

## 1. qpext 

当使用 Knative 部署时，一个 Pod 中通常包含两个容器：  
- **queue-proxy**：负责流量管理、健康检查、限流、超时和优雅关闭。  
- **kserve-container**：运行具体的模型预测工作负载。  

Prometheus 默认只能从单个端点抓取指标，而当一个 Pod 中有多个容器时，分别暴露的指标端口可能导致监控查询变得复杂。  
  
为了解决这个问题，**qpext（Queue Proxy EXTension）** 被引入：
- qpext 扩展了 Knative queue-proxy 的功能，使其可以同时查询（即聚合）来自 queue-proxy 容器和 kserve-container 的 Prometheus 指标，并在一个统一的端点输出聚合后的结果。
- 这样做可以让用户==只需要配置一个指标抓取端点，无需分开抓取多个容器的指标==，从而简化监控系统的部署与管理。

---

## 2. 主要功能和工作流程

> [!code]
>  `qpext/cmd/qpext/main.go`
### （1）解析与处理请求超时  
在代码中有 `getHeaderTimeout` 函数，其作用是解析 HTTP 请求头中的超时信息。例如，Prometheus 会通过 `X-Prometheus-Scrape-Timeout-Seconds` 头传递抓取时间：
````go
func getHeaderTimeout(timeout string) (time.Duration, error) {
	timeoutSeconds, err := strconv.ParseFloat(timeout, 64)
	if err != nil {
		return 0 * time.Second, err
	}
	return time.Duration(timeoutSeconds * 1e9), nil
}
````
解析得到的时间将用于为抓取请求设置上下文超时。

### （2）转发部分请求头  
`applyHeaders` 函数用于将来自原请求中的一些头（例如 "Accept"、"User-Agent"、抓取超时等）添加到新的抓取请求中，这样能保证原请求的部分信息传递下去：
````go
func applyHeaders(into http.Header, from http.Header, keys ...string) {
	for _, key := range keys {
		val := from.Get(key)
		if val != "" {
			into.Set(key, val)
		}
	}
}
````

### （3）获取 Serverless 标签  
在 qpext 中，通过 `getServerlessLabelVals` 获取了一组环境变量（如 SERVING_SERVICE、SERVING_CONFIGURATION、SERVING_REVISION）的值。这些值用于后续在指标中附加标签，以便在 Prometheus 中能按服务、配置和修订版本进行过滤和聚合：
````go
func getServerlessLabelVals() []string {
	var labelValues []string
	for _, envVar := range EnvVars {
		labelValues = append(labelValues, os.Getenv(envVar))
	}
	return labelValues
}
````

### （4）聚合指标  
qpext 的核心功能是将两个容器的指标聚合到一个端点上：
- 它通过 HTTP 客户端发起抓取请求（使用 `scrape` 函数）分别获取 queue-proxy 的指标和 kserve-container 的指标数据；
- 对于应用容器抓取到的指标，会调用 `scrapeAndWriteAppMetrics` 对指标进行预处理，比如：  
  - **添加 serverless 标签**：使用 `addServerlessLabels`（将环境变量中获取的标签添到每个指标）。
  - **指标清洗**：某些指标类型可能是 UNTYPED，通过 `sanitizeMetrics` 进行转换为 counter 或 gauge，使得 Prometheus 能正确处理这些指标。

````go
func (sc *ScrapeConfigurations) handleStats(w http.ResponseWriter, r *http.Request) {
	// ...
	if sc.QueueProxyPort != "" {
		queueProxyURL := getURL(sc.QueueProxyPort, sc.QueueProxyPath)
		// 抓取 queue-proxy 的指标
		if queueProxy, queueProxyCancel, _, err = scrape(queueProxyURL, r.Header, sc.logger); err != nil {
			sc.logger.Error("failed scraping queue proxy metrics", zap.Error(err))
		}
	}
	// 如果定义了应用端口，则抓取 kserve-container 的指标
	if sc.AppPort != "" {
		kserveContainerURL := getURL(sc.AppPort, sc.AppPath)
		if application, appCancel, contentType, err = scrape(kserveContainerURL, r.Header, sc.logger); err != nil {
			sc.logger.Error("failed scraping application metrics", zap.Error(err))
		}
	}
	// 设置输出格式为文本
	format := expfmt.FmtText
	w.Header().Set("Content-Type", string(format))
	// 先写入 queue-proxy 的指标，再处理应用指标
	if queueProxy != nil {
		_, err = io.Copy(w, queueProxy)
		if err != nil {
			sc.logger.Error("failed to scraping and writing queue proxy metrics", zap.Error(err))
		}
	}
	if application != nil {
		var parser expfmt.TextParser
		var mfs map[string]*io_prometheus_client.MetricFamily
		mfs, err = parser.TextToMetricFamilies(application)
		if err != nil {
			sc.logger.Error("error converting text to metric families", zap.Error(err))
		}
		if err = scrapeAndWriteAppMetrics(mfs, w, format, sc.logger); err != nil {
			sc.logger.Error("failed scraping and writing metrics", zap.Error(err))
		}
	}
}
````

在上面这段代码里，就做了下面几件事：
- 通过 `getURL` 构造两个抓取 URL（一个用于 queue-proxy，一个用于 kserve-container）。
- 分别调用 `scrape` 函数进行 HTTP 请求抓取。
- 抓取回来后将 queue-proxy 的原始指标直接写入响应，再进一步处理应用指标（添加标签、sanitize 后转换为文本格式）。

### （5）启动与服务部署  
在 main 函数中，qpext 作为一个独立的小应用启动一个 HTTP 服务器，在特定端口（通常是通过环境变量配置）上暴露聚合后的指标端点。用户或 Prometheus 抓取时，只访问该端点，就能获取到聚合好的指标。

````go
func main() {
	zapLogger := logger.InitializeLogger()
	mux := http.NewServeMux()
	ctx, cancel := context.WithCancel(context.Background())
	aggregateMetricsPort := os.Getenv(QueueProxyAggregatePrometheusMetricsPortEnvVarKey)
	sc := NewScrapeConfigs(
		zapLogger,
		QueueProxyMetricsPort,  // 默认队列代理 metrics 端口
		os.Getenv(KServeContainerPrometheusMetricsPortEnvVarKey), // kserve-container 的 metrics 端口
		os.Getenv(KServeContainerPrometheusMetricsPathEnvVarKey), // 指标 path
	)
	mux.HandleFunc(`/metrics`, sc.handleStats)
	l, err := net.Listen("tcp", fmt.Sprintf(":%v", aggregateMetricsPort))
	// 启动 stats server 和 sharedmain（队列代理主逻辑）
	...
}
````

---

