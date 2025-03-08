---
title: K8s-杂记
date: 2025-03-08
slug: blog-post-slug
tags:
  - Kubernetes
categories:
  - Blog
description: 描述
draft: false
state: "0"
---

## **1. 问题背景**
- **环境**：使用 `kind` 本地集群部署 Knative，服务 URL 为 `http://hello.knative-operator.knative.example.com`。
- **现象**：
  - `curl` 访问无输出，浏览器提示“网页解析失败”。
  - 服务状态显示就绪，但实际无法访问。
- **关键问题**：
  - `kind` 集群默认不支持 `LoadBalancer` 类型服务（`EXTERNAL-IP` 为 `<pending>`）。
  - 全局代理环境变量 `ALL_PROXY` 导致请求错误转发。


## **2. 排查过程**

### **2.1 检查服务状态**
```bash
kubectl get ksvc hello -n knative-operator
# 确认 READY 为 True
```

### **2.2 验证 DNS 配置**
```bash
cat /etc/hosts | grep knative.example.com
# 预期：10.96.68.192 *.knative.example.com
```

### **2.3 检查代理干扰**
```bash
# 清除代理变量：
unset ALL_PROXY HTTP_PROXY HTTPS_PROXY
curl -v http://hello.knative-operator.knative.example.com
```

### **2.4 分析网络结构**
- **集群内部 IP**：`istio-ingressgateway` 的 `CLUSTER-IP` 为 `10.96.68.192`。
- **Docker 容器 IP**：`172.18.0.2`（与集群网络隔离）。


## **3. 解决方案**

### **3.1 临时方案：端口转发**
1. **将集群服务端口映射到本地**：
   ```bash
   kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
   ```
2. **访问服务**：
   ```bash
   curl http://localhost:8080 -H "Host: hello.knative-operator.knative.example.com"
   ```

### **3.2 长期方案：配置 Kind 支持 LoadBalancer**
1. **创建 `kind-config.yaml`**：
   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   networking:
     loadBalancer:
       enabled: true
   ```
2. **重建集群**：
   ```bash
   kind delete cluster
   kind create cluster --config kind-config.yaml
   ```


## **4. 注意事项**
- **Kind 集群限制**：
  - 默认不支持 `LoadBalancer`，需通过 `NodePort` 或端口转发访问。
  - 服务 IP 为集群内部地址，需通过 `hosts` 文件或 DNS 配置解析。
- **代理干扰**：
  - 全局代理会影响本地请求，建议通过 `--noproxy` 或清除环境变量解决。
- **服务验证**：
  ```bash
  kubectl get pods -l serving.knative.dev/service=hello -n knative-operator
  # 确保所有 Pod 处于 Running 状态
  ```


## **5. 参考资料**
- [Knative 官方文档](https://knative.dev/docs/)
- [Kind 网络配置指南](https://kind.sigs.k8s.io/docs/user/loadbalancer/)


**标签**： #Knative #Kubernetes #Kind #Networking
