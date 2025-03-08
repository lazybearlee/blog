---
title: K8s-问题记录-ko私有仓库部署镜像拉取失败
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
## 问题背景
在使用 `ko` 工具搭建镜像并部署到 Kubernetes 集群（使用阿里云容器镜像服务）时，遇到 `kubectl wait` 命令超时的问题，具体错误信息为：
```
error: timed out waiting for the condition on pods/kserve-controller-manager-7cfdf8cb4c-5f22w
make: *** [Makefile:98: deploy-dev] Error 1
```
经检查，发现 Pod 处于 `ImagePullBackOff` 状态，表明镜像拉取失败。

## 排查过程

### 1. 查看 Pod 状态
执行命令：
```bash
kubectl get pods -l control-plane=kserve-controller-manager -n kserve
```
输出结果：
```
NAME                                         READY   STATUS             RESTARTS   AGE
kserve-controller-manager-7cfdf8cb4c-5f22w   1/2     ImagePullBackOff   0          7m46s
```
从输出可知，Pod 处于 `ImagePullBackOff` 状态，说明镜像拉取存在问题。

### 2. 查看详细错误信息
执行命令：
```bash
kubectl describe pod kserve-controller-manager-7cfdf8cb4c-5f22w -n kserve
```
输出中包含关键错误信息：
```
failed to pull and unpack image "crpi-h5pf7vq24twssgqd.cn-beijing.personal.cr.aliyuncs.com/lizhuo1/manager-12782ae64d3f5f3dbe4595b05a2cb78d@sha256:62cab9b00b4cd99dc60b6075ba83603a0e533c809f77ace29a382164fd924786": failed to resolve reference "crpi-h5pf7vq24twssgqd.cn-beijing.personal.cr.aliyuncs.com/lizhuo1/manager-12782ae64d3f5f3dbe4595b05a2cb78d@sha256:62cab9b00b4cd99dc60b6075ba83603a0e533c809f77ace29a382164fd924786": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```
此错误表明镜像拉取需要授权，可能是缺少镜像拉取 Secret 或配置不正确。

### 3. 尝试在 YAML 文件中添加 `imagePullSecrets`
在相关的 YAML 文件中添加 `imagePullSecrets` 配置，然后执行部署命令：
```bash
kubectl apply --server-side=true --force-conflicts -k config/overlays/development
```
出现错误信息：
```
Error from server: failed to create typed patch object (kserve/kserve-controller-manager; apps/v1, Kind=Deployment): .spec.template.imagePullSecrets: field not declared in schema
make: *** [Makefile:95: deploy-dev] Error 1
```
这是因为 `imagePullSecrets` 的位置不正确，正确的位置应该在 `spec.template.spec` 下。

## 解决方案：使用默认 ServiceAccount 绑定 Secret

- 如果在每个yaml文件中都加入secret会非常麻烦
- 而Kubernetes 中的 Pod 会默认使用所在命名空间的 `default` ServiceAccount。我们可以将镜像拉取 Secret 添加到 `default` ServiceAccount 中，这样该命名空间下所有使用 `default` ServiceAccount 的 Pod 都会自动继承这个 Secret，无需为每个 Pod 或 Deployment 单独配置。
### 1. 创建镜像拉取 Secret
执行命令创建 `aliyun-acr-secret`：
```bash
kubectl create secret docker-registry aliyun-acr-secret \
  --docker-server=crpi-h5pf7vq24twssgqd.cn-beijing.personal.cr.aliyuncs.com \
  --docker-username=your-aliyun-username \
  --docker-password=your-aliyun-password \
  -n kserve
```
其中 `your-aliyun-username` 和 `your-aliyun-password` 需替换为实际的阿里云账号和密码。

### 2. 查看当前 `default` ServiceAccount 配置
执行命令：
```bash
kubectl get serviceaccount default -n kserve -o yaml
```
输出结果显示当前 `default` ServiceAccount 的详细配置信息。

### 3. 将镜像拉取 Secret 添加到 `default` ServiceAccount
执行命令：
```bash
kubectl patch serviceaccount default -n kserve -p '{"imagePullSecrets": [{"name": "aliyun-acr-secret"}]}'
```
此命令将 `aliyun-acr-secret` 添加到 `kserve` 命名空间下的 `default` ServiceAccount 中。

### 4. 重新部署应用
再次执行部署命令：
```bash
kubectl apply --server-side=true --force-conflicts -k config/overlays/development
```
这次部署成功，Pod 能够正常拉取镜像并启动。

### 5. 验证部署结果
执行命令查看 Pod 状态：
```bash
kubectl get pods -l control-plane=kserve-controller-manager -n kserve
```
输出结果显示 Pod 处于 `Running` 状态，且 `READY` 字段显示为 `2/2`，表明应用已成功部署并正常运行。

