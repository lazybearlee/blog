---
title: 使用kind安装部署knative与istio
date: 2025-03-24
slug: blog-post-slug
tags:
  - Kubernetes
  - Knative
categories:
  - Blog
description: 描述
draft: false
state: "0"
---
作为云原生和Serverless领域的实践者，经常需要搭建Knative环境进行开发和测试。本文将介绍如何使用arkade工具快速安装istioctl、helm等工具，并通过kind创建一个本地Kubernetes集群，然后在其上部署Knative Serving、Istio和Cert-Manager。

## 环境准备

### 推荐工具安装

> 需要注意的坑是，arkade可能安装的不是最新版，最好去各个工具官网看一下版本
> 同时，kind的版本对集群k8s版本有影响，也需要注意

```shell
# 使用arkade一键安装所需工具
arkade get istioctl helm kubectl kind
```

### 创建Kind集群

```shell
kind create cluster
```

输出示例：
```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

## 安装Knative Serving

Knative Serving是构建Serverless工作负载的核心组件。

### 安装CRDs

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-crds.yaml
```

### 安装核心组件

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-core.yaml
```

## 安装Istio作为网络层

Istio提供了强大的服务网格能力，是Knative推荐的网络层实现。

### 使用istioctl安装Istio

```shell
istioctl install -y
```

输出示例：
```
        |\          
        | \         
        |  \        
        |   \      
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Istio core installed ⛵️                                                          
✔ Istiod installed 🧠                                                              
✔ Ingress gateways installed 🛬                                                    
✔ Installation complete                                                            
```

### 安装Knative Istio集成

```shell
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.17.0/net-istio.yaml
```

### 验证Istio Ingress Gateway

```shell
kubectl --namespace istio-system get service istio-ingressgateway
```

## 安装Cert-Manager

Cert-Manager为Knative提供了自动化的证书管理能力。

```shell
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
   cert-manager jetstack/cert-manager \
   --namespace cert-manager \
   --create-namespace \
   --version v1.16.1 \
   --set crds.enabled=true
   
echo "😀 Successfully installed Cert Manager"
```

## 环境验证

完成以上步骤后，就已经拥有了一个完整的Knative环境，可以开始部署Serverless应用了。

## 参考文档

- [Kind快速开始](https://kind.kubernetes.ac.cn/docs/user/quick-start/)
- [Knative Serving YAML安装指南](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/#install-a-networking-layer)
- [Istio官方文档](https://istio.io/latest/docs/)
- [Cert-Manager文档](https://cert-manager.io/docs/)