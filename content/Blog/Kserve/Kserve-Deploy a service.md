---
title: Kserve源码解析-梳理部署一个Sklearn服务的过程
date: 2025-03-11
slug: blog-post-slug
tags:
  - Kserve
categories:
  - Blog
description: 描述
draft: true
state: "0"
---
 
> 背景：部署Sklearn时发现...什么反应也没有，查看controller日志发现，predictor没能正确部署，于是干脆梳理一遍吧

## 从部署开始

按照官方文档，我们可用通过yaml文件部署一个 sklearn 的 inferenceService ，不过我们从源码也可以发现如下的 python 测试代码，也可以实现使用k8s client来创建一个 V1beta1InferenceService ：

```python
@pytest.mark.predictor
@pytest.mark.asyncio(scope="session")
async def test_sklearn_kserve(rest_v1_client):
    service_name = "isvc-sklearn"
    predictor = V1beta1PredictorSpec(
        min_replicas=1,
        sklearn=V1beta1SKLearnSpec(
            storage_uri="gs://kfserving-examples/models/sklearn/1.0/model",
            resources=V1ResourceRequirements(
                requests={"cpu": "50m", "memory": "128Mi"},
                limits={"cpu": "100m", "memory": "256Mi"},
            ),
        ),
    )

    isvc = V1beta1InferenceService(
        api_version=constants.KSERVE_V1BETA1,
        kind=constants.KSERVE_KIND_INFERENCESERVICE,
        metadata=client.V1ObjectMeta(
            name=service_name, namespace=KSERVE_TEST_NAMESPACE
        ),
        spec=V1beta1InferenceServiceSpec(predictor=predictor),
    )

    kserve_client = KServeClient(
        config_file=os.environ.get("KUBECONFIG", "~/.kube/config")
    )
    kserve_client.create(isvc)
    kserve_client.wait_isvc_ready(service_name, namespace=KSERVE_TEST_NAMESPACE)
    res = await predict_isvc(rest_v1_client, service_name, "./data/iris_input.json")
    assert res["predictions"] == [1, 1]
    kserve_client.delete(service_name, KSERVE_TEST_NAMESPACE)
```

当我们在集群中创建一个 V1beta1InferenceService 时，会发生什么呢？

## 控制器

当我们研读kserve源码的时候，会发现Kserve是使用kubebuilder开发的，实际上也就是k8s的operator开发。相关内容可以看看[Kubebuilder(1)-Get started](Blog/K8s/Kubebuilder(1)-Get%20started.md)。

在 `cmd/manager/main.go` 文件中，我们可以看到控制器的相关代码。而与 `InferenceService` 有关的主要是下面这两段：

```go
setupLog.Info("Setting up v1beta1 controller")
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: clientSet.CoreV1().Events("")})
if err = (&v1beta1controller.InferenceServiceReconciler{
	Client:    mgr.GetClient(),
	Clientset: clientSet,
	Log:       ctrl.Log.WithName("v1beta1Controllers").WithName("InferenceService"),
	Scheme:    mgr.GetScheme(),
	Recorder: eventBroadcaster.NewRecorder(
		mgr.GetScheme(), corev1.EventSource{Component: "v1beta1Controllers"}),
}).SetupWithManager(mgr, deployConfig, ingressConfig); err != nil {
	setupLog.Error(err, "unable to create controller", "v1beta1Controller", "InferenceService")
	os.Exit(1)
}
```

以及

```go
if err = ctrl.NewWebhookManagedBy(mgr).
	For(&v1beta1.InferenceService{}).
	WithDefaulter(&v1beta1.InferenceServiceDefaulter{}).
	WithValidator(&v1beta1.InferenceServiceValidator{}).
	Complete(); err != nil {
	setupLog.Error(err, "unable to create webhook", "webhook", "v1beta1")
	os.Exit(1)
}

if err = ctrl.NewWebhookManagedBy(mgr).
	For(&v1alpha1.LocalModelCache{}).
	WithValidator(&localmodelcache.LocalModelCacheValidator{Client: mgr.GetClient()}).
	Complete(); err != nil {
	setupLog.Error(err, "unable to create webhook", "webhook", "localmodelcache")
	os.Exit(1)
}
```

分别配置控制器和webhook准入。

`SetupWithManager` 方法实现了控制器的自适应注册机制，根据集群环境动态配置监听资源。

```go
// 1. 基础监听配置
ctrlBuilder := ctrl.NewControllerManagedBy(mgr).
    For(&v1beta1.InferenceService{}).    // 主资源
    Owns(&appsv1.Deployment{})           // 必需组件
```

```go
// 2. Knative 服务监听
if ksvcFound {
    ctrlBuilder = ctrlBuilder.Owns(&knservingv1.Service{})
}
```

```go
// 3. 入口配置
// Istio VirtualService
if vsFound && !ingressConfig.DisableIstioVirtualHost {
    ctrlBuilder = ctrlBuilder.Owns(&istioclientv1beta1.VirtualService{})
} 

// Gateway API vs Ingress
if ingressConfig.EnableGatewayAPI {
    ctrlBuilder = ctrlBuilder.Owns(&gatewayapiv1.HTTPRoute{})
} else {
    ctrlBuilder = ctrlBuilder.Owns(&netv1.Ingress{})
}
```
- 使用 `Owns()` 建立资源所有权关系
- 通过配置控制 Istio 和网关 API 的使用
- 错误处理和日志记录完善

然后让我们回到控制器的最主要方法—— `Reconcile` 。

## `Reconcile`

