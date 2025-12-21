# 多模态生成模型推理与调度 阅读清单（初版，可增量更新）

> 版本：v0.2（联网调研补充）  
> 说明：按主题分组，含优先级（P0/P1/P2）。清单为“候选+精选”，随着实践推进逐步勾选并沉淀笔记。本文中每条以 `[#RID]` 供学习计划引用。

---

## 总览与路线图

- [ ] [#OV-AWESOME] P1 Awesome ML Systems 教程索引（zhaochenyang20/Awesome-ML-SYS-Tutorial）  
  https://github.com/zhaochenyang20/Awesome-ML-SYS-Tutorial
- [ ] [#OV-PAPERS] P2 ML Systems 论文索引（byungsoo-oh/ml-systems-papers）  
  https://github.com/byungsoo-oh/ml-systems-papers
- [ ] [#OV-BLOG] P2 MLsys 博客整理  
  https://fazzie-key.cool/2023/02/21/MLsys/

---

## 生成模型推理基础

- [ ] [#GEN-PRIMER] P0 生成模型推理快速入门（自建笔记，覆盖内存结构/常见算子/典型路径）
- [ ] [#DIFF-CORE] P0 Diffusion/Latent Diffusion/SDXL/DiT/Rectified Flow/LCM 推理路径（论文/官方实现走读）
- [ ] [#VID-AUD] P1 视频与音频生成的推理特性（Open-Sora、CogVideoX、VideoCrafter2；AudioLDM2/BigVGAN）  
  https://github.com/showlab/Awesome-Video-Diffusion  
  https://huggingface.co/docs/diffusers/main/api/pipelines/audioldm2  
  https://github.com/NVIDIA/BigVGAN

---

## 推理部署与引擎

- [ ] [#RT-DEPLOY] P0 推理引擎部署总览（TensorRT / ONNX Runtime / PyTorch Inductor）
- [ ] [#RT-TRT-SD] P1 TensorRT 部署 Diffusion（经验/案例）  
  https://resources.nvidia.com/en-us-inference-contact-us/tensorrt-getting-started  
  https://www.baseten.co/blog/40-faster-stable-diffusion-xl-inference-with-nvidia-tensorrt/
- [ ] [#RT-ONNX] P1 ONNX Runtime 使用与优化（Stable Diffusion）  
  https://huggingface.co/docs/diffusers/en/optimization/onnx  
  https://onnxruntime.ai/docs/tutorials/csharp/stable-diffusion-csharp.html
- [ ] [#RT-TORCH] P2 PyTorch 2.x Inference（Inductor/Compile，侧重部署与内存）

---

## IO / 内存 / 加载

- [ ] [#IO-MEM] P0 safetensors + mmap 流式加载实践（HF 文档与实现走读）  
  https://huggingface.co/docs/safetensors/en/index  
  https://huggingface.co/docs/diffusers/main/using-diffusers/using_safetensors
- [ ] [#IO-GDS] P1 NVIDIA GPUDirect Storage 指南  
  https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html  
  https://docs.nvidia.com/gpudirect-storage/design-guide/index.html
- [ ] [#IO-PINNED] P1 Pinned 内存/多流拷贝/PCIe 重叠（CUDA 最佳实践）  
  https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/
- [ ] [#IO-ROBUST] P1 分块校验、失败重试、部分恢复策略（工程实践）

---

## CUDA / GPU 虚拟化与容器

- [ ] [#NV-STACK] P0 NVIDIA 容器/云原生栈（K8s、GPU Operator、MIG 支持）  
  https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html
- [ ] [#CUDA-MEM] P0 CUDA 内存管理（显存碎片、分配器、Stream 优先级）
- [ ] [#MIG-GUIDE] P1 NVIDIA MIG 用户指南  
  https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html

---

## Kubernetes / 调度 / GPU 设备

- [ ] [#K8S-GPU] P0 K8s GPU 设备与度量：NVIDIA Device Plugin / DCGM Exporter / GPU Operator  
  https://github.com/NVIDIA/k8s-device-plugin  
  https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/dcgm-exporter.html  
  https://github.com/NVIDIA/dcgm-exporter
- [ ] [#OBS] P0 观测：Prometheus + Grafana + DCGM（面板与告警）  
  https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/dcgm-exporter.html
- [ ] [#SCHED-POL] P1 资源画像/放置策略/混部案例（文档+生产实践）  
  https://volcano.sh/en/docs/  
  https://github.com/volcano-sh/volcano
- [ ] [#K8S-SCHED] P1 K8s 调度扩展（调度器框架/自定义调度），含 Volcano  
  https://volcano.sh/en/docs/
- [ ] [#MT-ISO] P1 多租户隔离与限流（Queue/优先级/配额）

---

## 工件与注册表

- [ ] [#ART-REG] P1 模型工件与注册表（版本、元数据、兼容矩阵；参考 MLflow/自研方案）  
  https://mlflow.org/docs/latest/ml/model-registry/

---

## 分布式与拓扑（按需）

- [ ] [#DIST-COMM] P2 分布式通信与零拷贝（NCCL/RDMA，推理注意事项）
- [ ] [#TOPO] P2 GPU/NUMA 拓扑亲和、NVLink 拓扑与放置

---

## 容量、SLA 与稳定性

- [ ] [#QUEUE] P1 朴素排队论在服务容量中的应用（M/M/k 直观用法）
- [ ] [#COST-REL] P1 成本-可靠性权衡与容量规划实践
- [ ] [#POSTMORTEM] P1 事故复盘与演练方法（Runbook/故障注入）
- [ ] [#AB-TEST] P2 A/B 与灰度评估（统计显著性与指标设计）

---

## 行业/前沿系统（取其思想，非 LLM 指标）

- [ ] [#IND-VLLM-OMNI] P2 vLLM-Omni（多模态与非自回归支持；借鉴内存/调度思想）  
  https://github.com/vllm-project/vllm-omni  
  https://blog.vllm.ai/2025/11/30/vllm-omni.html
- [ ] [#IND-SGLANG-DIFF] P1 SGLang Diffusion（扩展 SGLang 到 diffusion 的调度/加速）  
  https://lmsys.org/blog/2025-11-07-sglang-diffusion/  
  https://github.com/sgl-project/sglang
- [ ] [#IND-DIFFUSERS] P1 HuggingFace diffusers（管线、权重格式、调度器实现走读）  
  https://huggingface.co/docs/diffusers/index
- [ ] [#IND-COMFY] P2 ComfyUI/工作流图调度（算子图与缓存策略启发）

---

## 实用加速与系统论文（Diffusion 专向）

- [ ] [#DIFF-CACHE] P1 Approximate Caching for Efficiently Serving Text-to-Image Diffusion Models（NSDI’24）  
  https://www.usenix.org/system/files/nsdi24-agarwal-shubham.pdf
- [ ] [#DIFF-DIFFSERVE] P1 DiffServe: Efficiently Serving Text-to-Image Diffusion Models with Query-Aware Model Scaling（arXiv’24 → MLSys’25）  
  https://arxiv.org/abs/2411.15381
- [ ] [#DIFF-TRIDENT] P2 TridentServe: A Stage-level Serving System for Diffusion Pipelines（arXiv’25）  
  https://arxiv.org/abs/2510.02838

---

## 案例与实践集合

- [ ] [#IND-CASE] P1 生产侧实践与案例合辑（选读）  
  GPU Operator 实战综述：https://www.spectrocloud.com/blog/the-real-world-guide-to-the-nvidia-gpu-operator-for-kubernetes-ai  
  MIG on K8s（官方指南）：https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html  
  Diffusion 服务系统：[#DIFF-CACHE]、[#DIFF-DIFFSERVE]

---

## 备注与下一步

- 本清单将与《学习与工作结合计划》联动。阅读完成后请在此勾选并在计划文档中对应 Sprint 处补上 `[#RID]` 的链接与笔记位置。
- 如果后续获取到本地 `AI_Infra_Course_List` 的 Markdown 版本，将在本清单中映射其目录项到相应 `[#RID]`。

- [ ] [#FUTURE] P2 未来关注点与方向  
  vLLM-Omni：https://github.com/vllm-project/vllm-omni  
  SGLang Diffusion：https://lmsys.org/blog/2025-11-07-sglang-diffusion/  
  TridentServe：https://arxiv.org/abs/2510.02838
