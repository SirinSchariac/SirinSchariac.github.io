---
layout: 	post  				   
title: 		NVIDIA K8s DRA Driver				
subtitle:	GPU管理
date:       2026-01-28 				
author:     Sirin 						
header-img: img/sunset.jpg
catalog: 	true 				
tags:						
    - Kubernetes
    - golang
---

### 关于NVIDIA DRA Driver#gpu-kubelet-plugin

这里主要探讨GPU allocation side，即关于GPU资源动态分配的部分(`cmd/gpu-kubelet-plugin`)。另外一部分ComputeDomain端主要针对Multi-Node NVLink的拓扑与Pod间资源隔离实现。

对于`gpu-kubelet-plugin`，其核心作用是**在节点上，响应Kubelet的请求，为Pod准备和释放GPU资源。**整体的模块执行流程如下：

1. **启动和注册 (main.go)**
   - 用户（或部署脚本）在每个需要 GPU 的 K8s节点上启动 `gpu-kubelet-plugin` 进程。
   - `main.go` 中的代码执行创建一个 gRPC 服务器。
   - 插件通过 Unix Domain Socket 向 Kubelet 的插件管理器注册自己为一个 `dra.k8s.io/v1alpha3` 类型的插件，作为处理 `gpu.dra.nvidia.com` 这个驱动的插件。
2. **Kubelet 发起资源准备请求**
   - 当一个声明了需要 GPU 资源的 Pod 被调度到这个 Node 上时，Kubelet 会通过之前注册的 Socket 调用 `gpu-kubelet-plugin` 的 `NodePrepareResource` gRPC 方法。
   - 这个请求会携带一个 `ClaimUID`，它唯一标识了这次资源请求。
3. **插件处理资源准备 (plugin.go)**
   - `plugin.go` 中的 `NodePrepareResource` 方法被触发。
   - 它首先会从 `PreparedClaims` 自定义资源中获取该 `ClaimUID` 对应的具体 GPU 分配信息（比如分配了哪些 GPU 的 UUID）。这个信息是由另一个控制器（`dra-controller`）在调度时写入的。
   - 插件根据获取到的 GPU UUIDs，调用 `cdi/` 模块的功能。
4. **生成 CDI 设备规范 (cdi/)**
   - `cdi/` 模块的逻辑被调用，它的任务是为这次资源分配动态生成一个 CDI JSON Spec 文件。
   - 这个 JSON 文件会放在一个特定的目录（如 `/var/run/cdi/`），容器运行时会监控这个目录。
   - 文件内容会描述一个**唯一**的 CDI Device Name（例如 `gpu.dra.nvidia.com/claim-xxxxx=gpu-uuid-1,gpu-uuid-2`），并指明当容器引用这个设备名时，需要注入哪些设备节点（`/dev/nvidia0`）、库文件和环境变量。
5. **返回 CDI 句柄给 Kubelet**
   - CDI Spec 生成成功后，`plugin.go` 的 `NodePrepareResource` 方法会将这个唯一的 CDI Device Name 作为结果返回给 Kubelet。
6. **Kubelet 更新 Pod 并启动容器**
   - Kubelet 收到这个 CDI Device Name 后，会将其注入到 Pod 的运行时规范（Runtime Spec）中。
   - 当容器运行时（如 containerd）启动该 Pod 的容器时，它会看到这个 CDI 设备引用，然后根据 CDI 规范去查找对应的 JSON Spec 文件，并按照文件中的描述来正确配置容器的运行环境（即把 GPU “插”进容器里）。
7. **资源释放**
   - 当 Pod 结束后，Kubelet 会调用插件的 `NodeUnprepareResource` 方法。
   - `plugin.go` 中的 `NodeUnprepareResource` 逻辑会触发，它会调用 `cdi/` 模块去删除之前为该 Pod 生成的 CDI JSON Spec 文件，从而完成资源的清理。

#### 该插件能否实现动态修改Pod运行时资源配置？

不能，该插件创建的CDI Spec是在容器启动时刻被读取并注册到容器运行时的配置中。

目前KEP中有一个关于Pod资源动态修改的[Proposal](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1287-in-place-update-pod-resources)，不过没有提到关于GPU资源，主要是CPU和内存资源。
