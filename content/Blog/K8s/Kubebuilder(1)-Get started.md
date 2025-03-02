---
title: Kubebuilder(1)-Get started
date: 2025-03-02
slug: blog-post-slug
tags:
  - Serverless
  - K8sOperator
  - Kubebuilder
categories:
  - Blog
description: 描述
summary: 摘要
draft: true
---

### 入门指南
我们将创建一个示例项目，让你了解它是如何工作的。这个示例将：
- 协调一个Memcached自定义资源（CR），它代表在集群上部署/管理的Memcached实例。
- 使用Memcached镜像创建一个Deployment。
- 不允许实例数量超过自定义资源（CR）中定义的规模，并且会应用该限制。
- 更新Memcached CR的状态。

#### 为什么使用Operator？
通过遵循Operator模式，不仅可以提供所有预期的资源，还可以在运行时以编程方式动态管理这些资源。为了说明这一概念，设想一下，如果有人意外更改了配置或误删了资源；在这种情况下，Operator可以在无需人工干预的情况下修复问题。

#### 按部就班还是直接跳跃？
请注意，本教程的大部分内容是由可运行项目的Go文件生成的，这些文件位于本书的源目录：`docs/book/src/getting-started/testdata/project`。

#### 创建项目
首先，为你的项目创建并进入一个目录。然后，使用`kubebuilder`进行初始化：
```bash
mkdir $GOPATH/memcached-operator
cd $GOPATH/memcached-operator
kubebuilder init --domain=example.com
```
**在GOPATH中开发**：如果你的项目在`GOPATH`中初始化，隐式调用的`go mod init`会为你插入模块路径。否则，必须设置`--repo=<module path>`。如果不熟悉模块系统，请阅读Go模块博客文章。

#### 创建Memcached API（CRD）
接下来，我们将创建负责在集群上部署和管理Memcached实例的API。
```bash
kubebuilder create api --group cache --version v1alpha1 --kind Memcached
```
**理解APIs**：这个命令的主要目的是为Memcached类型生成自定义资源（CR）和自定义资源定义（CRD）。它创建了一个API，组为`cache.example.com`，版本为`v1alpha1`，唯一标识了Memcached类型的新CRD。通过利用Kubebuilder工具，我们可以为这些平台定义代表我们解决方案的API和对象。在这个例子中，我们只添加了一种类型的资源，但根据需要，我们可以拥有任意数量的组（Groups）和类型（Kinds）。为了便于理解，可以将CRD看作是我们自定义对象的定义，而CR则是它们的实例。请务必查看“Groups and Versions and Kinds, oh my!”。
**注解**：CRD（Custom Resource Definition）即自定义资源定义，是Kubernetes扩展机制的一部分，允许用户定义自己的资源类型，以满足特定的业务需求。通过定义CRD，用户可以在Kubernetes集群中使用自定义的资源，就像使用内置资源（如Pod、Deployment等）一样。例如，在这个示例中，通过`kubebuilder create api`命令创建的Memcached的CRD，定义了Memcached资源的结构和规范。

#### 定义我们的API
**定义Specs**：现在，我们将定义集群上每个Memcached资源实例可以采用的值。在这个例子中，我们将允许通过以下方式配置实例数量：
```go
type MemcachedSpec struct {
    ...
    Size int32 `json:"size,omitempty"`
}
```
**创建状态定义**：我们还希望跟踪管理Memcached CR时所执行操作的状态。这使我们能够验证自定义资源对我们自己API的描述，并确定一切是否成功执行，或者是否遇到任何错误，这与我们处理Kubernetes API中的任何资源的方式类似。
```go
// MemcachedStatus定义了Memcached的观察状态
type MemcachedStatus struct {
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
}
```
**状态条件**：Kubernetes已经建立了约定，因此我们在这里使用状态条件。我们希望我们的自定义API和控制器的行为类似于Kubernetes资源及其控制器，遵循这些标准以确保一致和直观的体验。请务必查看：Kubernetes API Conventions。
**标记和验证**：此外，我们希望验证添加到自定义资源中的值，以确保它们是有效的。为了实现这一点，我们将使用标记，如`+kubebuilder:validation:Minimum=1`。

现在，来看我们完整的示例：
```go
// 路径：../getting-started/testdata/project/api/v1alpha1/memcached_types.go
// 版权声明：Apache License
// Copyright 2025 The Kubernetes authors.
// 许可协议：Licensed under the Apache License, Version 2.0 (the “License”); you may not use this file except in compliance with the License. You may obtain a copy of the License at
// http://www.apache.org/licenses/LICENSE-2.0
// 免责声明：Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

// 包声明
package v1alpha1

// 导入必要的包
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// 注意：需要编辑此文件！这是供你使用的脚手架代码！
// 注意：json标签是必需的。你添加的任何新字段都必须有json标签，以便字段能够被序列化。
// MemcachedSpec定义了Memcached的期望状态
type MemcachedSpec struct {
    // 插入其他Spec字段 - 集群的期望状态
    // 重要提示：修改此文件后运行“make”以重新生成代码

    // Size定义了Memcached实例的数量
    // 以下标记将使用OpenAPI v3模式来验证该值
    // 更多信息：https://book.kubebuilder.io/reference/markers/crd-validation.html
    //+kubebuilder:validation:Minimum=1
    //+kubebuilder:validation:Maximum=3
    //+kubebuilder:validation:ExclusiveMaximum=false
    Size int32 `json:"size,omitempty"`
}

// MemcachedStatus定义了Memcached的观察状态
type MemcachedStatus struct {
    // 表示对Memcached当前状态的观察结果。
    // Memcached.status.conditions.type包括：“Available”、“Progressing”和“Degraded”
    // Memcached.status.conditions.status取值为True、False、Unknown之一。
    // Memcached.status.conditions.reason的值应该是驼峰式命名的字符串，特定条件类型的生产者可以定义该字段的预期值和含义，以及这些值是否被视为保证的API。
    // Memcached.status.conditions.Message是一个人类可读的消息，指示有关转换的详细信息。
    // 更多信息：https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
// Memcached是memcacheds API的模式
type Memcached struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MemcachedSpec   `json:"spec,omitempty"`
    Status MemcachedStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true
// MemcachedList包含Memcached的列表
type MemcachedList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Memcached `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Memcached{}, &MemcachedList{})
}
```

#### 使用Specs和验证生成清单文件
要生成所有必需的文件：
- 运行`make generate`，在`api/v1alpha1/zz_generated.deepcopy.go`中创建DeepCopy实现。
- 然后，运行`make manifests`，在`config/crd/bases`下生成CRD清单文件，并在`config/crd/samples`下生成一个示例。
这两个命令分别使用`controller-gen`并带有不同的标志来生成代码和清单文件。

```yaml
# 路径：config/crd/bases/cache.example.com_memcacheds.yaml：我们的Memcached CRD
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.17.2
  name: memcacheds.cache.example.com
spec:
  group: cache.example.com
  names:
    kind: Memcached
    listKind: MemcachedList
    plural: memcacheds
    singular: memcached
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: Memcached是memcacheds API的模式。
        properties:
          apiVersion:
            description: |-
              APIVersion定义了此对象表示的版本化模式。
              服务器应将识别的模式转换为最新的内部值，并可能拒绝不识别的值。
              更多信息：https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
            type: string
          kind:
            description: |-
              Kind是一个字符串值，表示此对象所代表的REST资源。
              服务器可以从客户端提交请求的端点推断出该值。
              不能更新。
              采用驼峰式命名（CamelCase）。
              更多信息：https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
            type: string
          metadata:
            type: object
          spec:
            description: MemcachedSpec定义了Memcached的期望状态。
            properties:
              size:
                description: |-
                  Size定义了Memcached实例的数量
                  以下标记将使用OpenAPI v3模式来验证该值
                  更多信息：https://book.kubebuilder.io/reference/markers/crd-validation.html
                format: int32
                maximum: 3
                minimum: 1
                type: integer
            type: object
          status:
            description: MemcachedStatus定义了Memcached的观察状态。
            properties:
              conditions:
                items:
                  description: Condition包含此API资源当前状态的一个方面的详细信息。
                  properties:
                    lastTransitionTime:
                      description: lastTransitionTime是条件从一种状态转换到另一种状态的最后时间。
                        这应该是基础条件发生变化的时间。如果不知道，则使用API字段更改的时间是可以接受的。
                      format: date-time
                      type: string
                    message:
                      description: message是一个人类可读的消息，指示有关转换的详细信息。
                        这可能是一个空字符串。
                      maxLength: 32768
                      type: string
                    observedGeneration:
                      description: observedGeneration表示设置条件所基于的.metadata.generation。例如，如果.metadata.generation当前为12，但.status.conditions[x].observedGeneration为9，则该条件相对于实例的当前状态已过时。
                      format: int64
                      minimum: 0
                      type: integer
                    reason:
                      description: reason包含一个编程标识符，指示条件最后一次转换的原因。
                        特定条件类型的生产者可以定义该字段的预期值和含义，以及这些值是否被视为保证的API。该值应该是驼峰式命名的字符串。此字段不能为空。
                      maxLength: 1024
                      minLength: 1
                      pattern: ^[A-Za-z]([A-Za-z0-9_,:]*[A-Za-z0-9_])?$
                      type: string
                    status:
                      description: 条件的状态，取值为True、False、Unknown之一。
                      enum:
                      - "True"
                      - "False"
                      - Unknown
                      type: string
                    type:
                      description: 条件的类型，采用驼峰式命名（CamelCase）或以foo.example.com/CamelCase的格式。
                      maxLength: 316
                      pattern: ^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])$
                      type: string
                  required:
                  - lastTransitionTime
                  - message
                  - reason
                  - status
                  - type
                  type: object
                type: array
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
```
**自定义资源示例**：位于`config/samples`目录下的清单文件用作可以应用到集群的自定义资源的示例。在这个特定示例中，通过将给定的资源应用到集群，我们将生成一个具有单个实例规模的Deployment（见`size: 1`）。
```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  labels:
    app.kubernetes.io/name: project
    app.kubernetes.io/managed-by: kustomize
  name: memcached-sample
spec:
  # 待办事项（用户）：编辑以下值，确保集群上的Pod/实例数量符合要求
  size: 1
```

#### 协调过程
简单来说，Kubernetes的工作方式是允许我们声明系统的期望状态，然后其控制器持续观察集群并采取行动，以确保实际状态与期望状态一致。对于我们的自定义API和控制器，这个过程是类似的。请记住，我们正在扩展Kubernetes的行为及其API，以满足我们的特定需求。
在我们的控制器中，我们将实现一个协调过程。本质上，协调过程就像一个循环，持续检查条件并执行必要的操作，直到达到期望状态。这个过程将一直运行，直到系统中的所有条件都与我们实现中定义的期望状态一致。
下面是一个伪代码示例来说明这一点：
```go
reconcile App {
  // 检查是否存在该应用的Deployment，如果不存在，则创建一个
  // 如果有错误，则从协调的开头重新开始
  if err != nil {
    return reconcile.Result{}, err
  }

  // 检查是否存在该应用的Service，如果不存在，则创建一个
  // 如果有错误，则从协调的开头重新开始
  if err != nil {
    return reconcile.Result{}, err
  }

  // 查找Database CR/CRD
  // 检查Database Deployment的副本数量
  // 如果deployment.replicas数量与cr.size不匹配，则更新它
  // 然后，从协调的开头重新开始。例如，通过返回`reconcile.Result{Requeue: true}, nil`。
  if err != nil {
    return reconcile.Result{Requeue: true}, nil
  }
  ...

  // 如果在循环结束时：
  // 所有操作都成功执行，并且协调可以停止
  return reconcile.Result{}, nil
}
```
**返回选项**：以下是一些可能的返回选项，用于重新启动协调：
- 带有错误：`return ctrl.Result{}, err`
- 没有错误：`return ctrl.Result{Requeue: true}, nil`
- 因此，要停止协调，使用：`return ctrl.Result{}, nil`
- 在X时间后再次协调：`return ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())}, nil`
**在我们的示例上下文中**：当我们的示例自定义资源（CR）应用到集群时（即`kubectl apply -f config/sample/cache_v1alpha1_memcached.yaml`），我们希望确保为我们的Memcached镜像创建一个Deployment，并且它与CR中定义的副本数量相匹配。
为了实现这一点，我们首先需要实现一个操作，检查集群上是否已经存在我们Memcached实例的Deployment。如果不存在，控制器将相应地创建Deployment。因此，我们的协调过程必须包括一个操作，以确保持续维护这个期望状态。这个操作将包括：
```go
// 检查Deployment是否已经存在，如果不存在则创建一个新的
found := &appsv1.Deployment{}
err = r.Get(ctx, types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, found)
if err != nil && apierrors.IsNotFound(err) {
    // 定义一个新的Deployment
    dep := r.deploymentForMemcached()
    // 在集群上创建Deployment
    if err = r.Create(ctx, dep); err != nil {
        log.Error(err, "Failed to create new Deployment",
        "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
        return ctrl.Result{}, err
    }
    ...
}
```
接下来，请注意 `deploymentForMemcached()` 函数需要定义并返回应该在集群上创建的Deployment。这个函数应该使用必要的规范构造Deployment对象，如下例所示：
```go
dep := &appsv1.Deployment{
    Spec: appsv1.DeploymentSpec{
        Replicas: &replicas,
        Template: corev1.PodTemplateSpec{
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{{
                    Image:           "memcached:1.6.26-alpine3.19",
                    Name:            "memcached",
                    ImagePullPolicy: corev1.PullIfNotPresent,
                    Ports: []corev1.ContainerPort{{
                        ContainerPort: 11211,
                        Name:          "memcached",
                    }},
                    Command: []string{"memcached", "--memory-limit=64", "-o", "modern", "-v"},
                }},
            },
        },
    },
}
```
此外，我们需要实现一种机制来验证集群上Memcached副本的数量是否与自定义资源（CR）中指定的期望数量相匹配。如果存在差异，协调过程必须更新集群以确保一致性。这意味着每当在集群上创建或更新Memcached类型的CR时，控制器将持续协调状态，直到实际副本数量与期望数量匹配。以下示例说明了这个过程：
```go
...
size := memcached.Spec.Size
if *found.Spec.Replicas != size {
    found.Spec.Replicas = &size
    if err = r.Update(ctx, found); err != nil {
        log.Error(err, "Failed to update Deployment",
            "Deployment.Namespace", found.Namespace, "Deployment.Name", found.Name)
        return ctrl.Result{}, err
    }
...
}
```
现在，你可以查看负责管理Memcached类型自定义资源的完整控制器。这个控制器确保集群中维持期望状态，确保我们的Memcached实例继续按照用户指定的副本数量运行。
```go
// 路径：internal/controller/memcached_controller.go：我们的控制器实现
/*
Copyright 2025 The Kubernetes authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
package controller

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/utils/ptr"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    cachev1alpha1 "example.com/memcached/api/v1alpha1"
)

// 定义管理状态条件的常量
const (
    // typeAvailableMemcached代表Deployment协调的状态
    typeAvailableMemcached = "Available"
)

// MemcachedReconciler协调一个Memcached对象
type MemcachedReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
// Reconcile是Kubernetes主要协调循环的一部分，旨在使集群的当前状态更接近期望状态。
// 控制器的协调循环必须是幂等的。通过遵循Operator模式，你将创建提供协调函数的控制器，
// 该函数负责同步资源，直到集群达到期望状态。违反此建议将违背controller - runtime的设计原则，
// 并可能导致意外后果，例如资源卡住并需要手动干预。
// 更多信息：
// - 关于Operator模式：https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
// - 关于控制器：https://kubernetes.io/docs/concepts/architecture/controller/
//
// 更多详细信息，请查看Reconcile及其Result：
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.20.2/pkg/reconcile
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 获取Memcached实例
    // 目的是检查集群上是否应用了Memcached类型的自定义资源，如果没有则返回nil停止协调
    memcached := &cachev1alpha1.Memcached{}
    err := r.Get(ctx, req.NamespacedName, memcached)
    if err != nil {
        if apierrors.IsNotFound(err) {
            // 如果找不到自定义资源，通常意味着它已被删除或未创建
            // 这样，我们将停止协调
            log.Info("memcached resource not found. Ignoring since object must be deleted")
            return ctrl.Result{}, nil
        }
        // 读取对象时出错 - 重新排队请求
        log.Error(err, "Failed to get memcached")
        return ctrl.Result{}, err
    }

    // 如果没有可用状态，将状态设置为Unknown
    if len(memcached.Status.Conditions) == 0 {
        meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached, Status: metav1.ConditionUnknown, Reason: "Reconciling", Message: "Starting reconciliation"})
        if err = r.Status().Update(ctx, memcached); err != nil {
            log.Error(err, "Failed to update Memcached status")
            return ctrl.Result{}, err
        }

        // 更新状态后重新获取Memcached自定义资源
        // 以便获取集群上资源的最新状态，避免出现
        // “the object has been modified, please apply your changes to the latest version and try again”错误，
        // 如果在后续操作中再次更新，该错误会重新触发协调
        if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
            log.Error(err, "Failed to re-fetch memcached")
            return ctrl.Result{}, err
        }
    }

    // 检查Deployment是否已经存在，如果不存在则创建一个新的
    found := &appsv1.Deployment{}
    err = r.Get(ctx, types.NamespacedName{Name: memcached.Name, Namespace: memcached.Namespace}, found)
    if err != nil && apierrors.IsNotFound(err) {
        // 定义一个新的Deployment
        dep, err := r.deploymentForMemcached(memcached)
        if err != nil {
            log.Error(err, "Failed to define new Deployment resource for Memcached")

            // 以下实现将更新状态
            meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached,
                Status: metav1.ConditionFalse, Reason: "Reconciling",
                Message: fmt.Sprintf("Failed to create Deployment for the custom resource (%s): (%s)", memcached.Name, err)})

            if err := r.Status().Update(ctx, memcached); err != nil {
                log.Error(err, "Failed to update Memcached status")
                return ctrl.Result{}, err
            }

            return ctrl.Result{}, err
        }

        log.Info("Creating a new Deployment",
            "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
        if err = r.Create(ctx, dep); err != nil {
            log.Error(err, "Failed to create new Deployment",
                "Deployment.Namespace", dep.Namespace, "Deployment.Name", dep.Name)
            return ctrl.Result{}, err
        }

        // Deployment创建成功
        // 我们将重新排队协调，以确保状态并继续进行下一步操作
        return ctrl.Result{RequeueAfter: time.Minute}, nil
    } else if err != nil {
        log.Error(err, "Failed to get Deployment")
        // 返回错误，重新触发协调
        return ctrl.Result{}, err
    }

    // CRD API定义Memcached类型有一个MemcachedSpec.Size字段
    // 用于将Deployment实例的数量设置为集群上的期望状态。
    // 因此，以下代码将确保Deployment的规模与我们正在协调的自定义资源的Size规范中定义的规模相同。
    size := memcached.Spec.Size
    if *found.Spec.Replicas != size {
        found.Spec.Replicas = &size
        if err = r.Update(ctx, found); err != nil {
            log.Error(err, "Failed to update Deployment",
                "Deployment.Namespace", found.Namespace, "Deployment.Name", found.Name)

            // 更新状态前重新获取Memcached自定义资源
            // 以便获取集群上资源的最新状态，避免出现
            // “the object has been modified, please apply your changes to the latest version and try again”错误，
            // 该错误会重新触发协调
            if err := r.Get(ctx, req.NamespacedName, memcached); err != nil {
                log.Error(err, "Failed to re-fetch memcached")
                return ctrl.Result{}, err
            }

            // 以下实现将更新状态
            meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached,
                Status: metav1.ConditionFalse, Reason: "Resizing",
                Message: fmt.Sprintf("Failed to update the size for the custom resource (%s): (%s)", memcached.Name, err)})

            if err := r.Status().Update(ctx, memcached); err != nil {
                log.Error(err, "Failed to update Memcached status")
                return ctrl.Result{}, err
            }

            return ctrl.Result{}, err
        }

        // 更新规模后重新排队协调
        // 以便在更新前确保我们拥有资源的最新状态，也有助于确保集群上的期望状态
        return ctrl.Result{Requeue: true}, nil
    }

    // 以下实现将更新状态
    meta.SetStatusCondition(&memcached.Status.Conditions, metav1.Condition{Type: typeAvailableMemcached,
        Status: metav1.ConditionTrue, Reason: "Reconciling",
        Message: fmt.Sprintf("Deployment for custom resource (%s) with %d replicas created successfully", memcached.Name, size)})

    if err := r.Status().Update(ctx, memcached); err != nil {
        log.Error(err, "Failed to update Memcached status")
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

// SetupWithManager将控制器与Manager进行设置
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1alpha1.Memcached{}).
        Owns(&appsv1.Deployment{}).
        Named("memcached").
        Complete(r)
}

// deploymentForMemcached返回一个Memcached Deployment对象
func (r *MemcachedReconciler) deploymentForMemcached(
    memcached *cachev1alpha1.Memcached) (*appsv1.Deployment, error) {
    replicas := memcached.Spec.Size
    image := "memcached:1.6.26-alpine3.19"

    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      memcached.Name,
            Namespace: memcached.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app.kubernetes.io/name": "project"},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app.kubernetes.io/name": "project"},
                },
                Spec: corev1.PodSpec{
                    SecurityContext: &corev1.PodSecurityContext{
                        RunAsNonRoot: ptr.To(true),
                        SeccompProfile: &corev1.SeccompProfile{
                            Type: corev1.SeccompProfileTypeRuntimeDefault,
                        },
                    },
                    Containers: []corev1.Container{{
                        Image:           image,
                        Name:            "memcached",
                        ImagePullPolicy: corev1.PullIfNotPresent,
                        // 确保容器具有严格的上下文
                        // 更多信息：https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted
                        SecurityContext: &corev1.SecurityContext{
                            RunAsNonRoot:             ptr.To(true),
                            RunAsUser:                ptr.To(int64(1001)),
                            AllowPrivilegeEscalation: ptr.To(false),
                            Capabilities: &corev1.Capabilities{
                                Drop: []corev1.Capability{
                                    "ALL",
                                },
                            },
                        },
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: 11211,
                            Name:          "memcached",
                        }},
                        Command: []string{"memcached", "--memory-limit=64", "-o", "modern", "-v"},
                    }},
                },
            },
        },
    }

    // 设置Deployment的ownerRef
    // 更多信息：https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/
    if err := ctrl.SetControllerReference(memcached, dep, r.Scheme); err != nil {
        return nil, err
    }
    return dep, nil
}
```
### 深入研究控制器实现
#### 设置Manager以监控资源
核心思想是监控对控制器重要的资源。当控制器感兴趣的资源发生变化时，Watch机制会触发控制器的协调循环，确保资源的实际状态与控制器逻辑中定义的期望状态一致。
注意我们是如何配置Manager来监控诸如Memcached类型自定义资源（CR）的创建、更新或删除事件，以及控制器管理和拥有的Deployment的任何变化：
```go
// SetupWithManager将控制器与Manager进行设置
// 也会监控Deployment以确保其在集群中的期望状态
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        // 监控Memcached自定义资源，每当它被创建、更新或删除时触发协调
        For(&cachev1alpha1.Memcached{}).
        // 监控由Memcached控制器管理的Deployment。如果此控制器拥有和管理的Deployment发生任何变化，
        // 它将触发协调，确保集群状态与期望状态一致
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```
#### 但是，Manager如何知道哪些资源是由它所拥有的呢？
我们不希望控制器监控集群上的任何Deployment并触发我们的协调循环。相反，我们只希望在运行我们Memcached实例的特定Deployment发生变化时触发协调。例如，如果有人意外删除了我们的Deployment或更改了副本数量，我们希望触发协调以确保它恢复到期望状态。
Manager知道要观察哪个Deployment是因为我们设置了`ownerRef`（所有者引用）：
```go
if err := ctrl.SetControllerReference(memcached, dep, r.Scheme); err != nil {
    return nil, err
}
```
**`ownerRef`和级联事件**：`ownerRef`不仅对于我们观察特定资源的变化至关重要，而且如果我们从集群中删除Memcached自定义资源（CR），我们希望由它所拥有的所有资源也能在级联事件中自动删除。
这确保了在删除父资源（Memcached CR）时，所有相关资源（如Deployment、Service等）也会被清理，从而维护集群状态的整洁和一致性。
#### 授予权限
确保控制器具有管理其资源所需的权限（即创建、获取、更新和列出）非常重要。
现在，基于角色的访问控制（RBAC）权限是通过RBAC标记进行配置的，这些标记用于生成和更新`config/rbac/`目录中的清单文件。这些标记可以在每个控制器的`Reconcile()`方法中找到（并且应该在其中定义），看看在我们的示例中是如何实现的：
```go
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
```
上述这些标记指定了我们的控制器（`MemcachedReconciler`）所需的权限。

- `cache.example.com` 组的 `memcacheds` 资源：控制器需要能够获取（`get`）、列出（`list`）、监视（`watch`）、创建（`create`）、更新（`update`）、修补（`patch`）和删除（`delete`）这些自定义资源（`Memcached`）。
- `cache.example.com` 组的 `memcacheds/status` 资源：用于获取、更新和修补 `Memcached` 资源的状态。这对于控制器报告资源的实际状态以及与期望状态的比较非常重要。
- `cache.example.com` 组的 `memcacheds/finalizers` 资源：允许更新最终izers。最终izers 是一种机制，用于确保在资源被删除之前执行某些清理操作。
- `core` 组的 `events` 资源：允许创建和修补事件。事件用于在集群中记录重要的操作和状态变化，控制器可以使用它们来传达有关其活动的信息。
- `apps` 组的 `deployments` 资源：因为我们的控制器管理 `Memcached` 资源的 `Deployment`，所以它需要对 `Deployment` 资源具有获取、列出、监视、创建、更新、修补和删除的权限。
- `core` 组的 `pods` 资源：由于 `Deployment` 管理 `Pod`，控制器可能需要获取、列出和监视 `Pod`，以便了解 `Memcached` 实例的运行状况和状态。

### 生成 RBAC 清单
运行以下命令来生成并更新 `config/rbac` 目录中的 RBAC 清单文件：
```bash
make rbac
```
此命令将根据在控制器代码中定义的 `kubebuilder:rbac` 标记，生成必要的 RBAC 角色、角色绑定和集群角色绑定。

### 部署到集群
要将我们的操作符（Operator）部署到 Kubernetes 集群，请运行以下命令：
```bash
make install deploy
```
`make install` 命令将在集群中安装自定义资源定义（CRD），创建我们定义的 `Memcached` 资源的模式。

`make deploy` 命令将构建操作符的容器镜像，并将其部署到集群中。这将启动我们的控制器，开始监视 `Memcached` 资源并协调它们的状态。

### 测试操作符
我们可以通过创建一个 `Memcached` 自定义资源的实例来测试操作符。使用以下示例 `Memcached` 资源清单：
```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  name: my-memcached
  namespace: default
spec:
  size: 3
```
将上述内容保存为一个 YAML 文件（例如 `memcached.yaml`），然后使用 `kubectl` 应用它：
```bash
kubectl apply -f memcached.yaml
```
这将在集群中创建一个具有 3 个副本的 `Memcached` 实例。操作符的控制器将检测到这个新资源，并开始协调以确保创建相应的 `Deployment` 并具有正确的副本数量。

你可以使用以下命令检查 `Memcached` 资源的状态：
```bash
kubectl get memcached my-memcached
```
以及检查 `Deployment` 的状态：
```bash
kubectl get deployments my-memcached
```
如果一切正常，你应该看到 `Memcached` 资源的状态反映了它的可用状态，并且 `Deployment` 具有 3 个运行中的副本。

### 清理
要清理并删除我们在集群中部署的所有内容，可以运行以下命令：
```bash
make undeploy uninstall
```
`make undeploy` 命令将删除操作符的部署和相关资源，例如服务帐户和角色绑定。

`make uninstall` 命令将删除我们安装的自定义资源定义（CRD），从集群中完全删除 `Memcached` 资源的模式。

这样，你就完成了一个基本的 Kubernetes 操作符的创建、部署和测试过程，该操作符管理自定义的 `Memcached` 资源。

**注解**：
- **Kubernetes 操作符（Operator）**：是一种基于 Kubernetes API 扩展机制构建的应用程序，用于管理复杂的有状态应用程序。它通过自定义资源（CR）和控制器的组合，实现对应用程序的自动化部署、配置和管理。操作符可以理解应用程序的领域逻辑，并根据自定义资源的定义来确保应用程序的状态符合期望。
- **协调（Reconciliation）**：在 Kubernetes 和操作符的上下文中，协调是指控制器不断检查资源的实际状态与期望状态之间的差异，并采取必要的行动来使实际状态与期望状态一致的过程。例如，在我们的 `Memcached` 操作符中，控制器会检查 `Memcached` 自定义资源中指定的副本数量与实际 `Deployment` 中的副本数量是否一致，如果不一致则进行调整。
- **Kubernetes 资源的所有权（Ownership）**：通过设置 `ownerRef`，我们可以定义资源之间的所有权关系。当一个资源（所有者）被删除时，其拥有的所有资源（依赖者）可以根据配置自动被删除。这有助于维护集群中资源的一致性和整洁性，避免孤立资源的存在。

希望以上内容对你撰写关于 K8s 和 K8s Operator 的博客有所帮助。如果你还有其他需求，请随时告诉我。 
```go
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
```