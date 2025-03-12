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

当我们研读kserve源码的时候，会发现