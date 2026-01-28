---
layout: 	post  				   
title: 		NVIDIA K8s Device Plugin				
subtitle:	GPU管理
date:       2026-01-28 				
author:     Sirin 						
header-img: img/sunset.jpg
catalog: 	true 				
tags:						
    - Kubernetes
    - golang
---

### 关于NVIDIA Device Plugin

主要的作用包括

- 向集群上的每个节点暴露GPU数量，将GPU作为拓展资源(`nvidia.com/gpu`)注册
- 监控GPU状态
- 在K8s集群上运行使用GPU的容器

与K8s框架集成的示意图如下：

```
┌────────────────────────────────────────────┐
│                   Kubernetes               │
│  ┌─────────────┐    ┌──────────────┐       │
│  │   Scheduler │    │   Kubelet    │       │
│  └──────┬──────┘    └──────┬───────┘       │
│         │                  │               │
│  ┌──────▼──────┐    ┌──────▼──────────┐    │
│  │  API Server │    │ Device Plugin   │    │
│  │             │    │    Interface    │    │
│  └─────────────┘    └────────┬────────┘    │
└──────────────────────────────│─────────────┘
                               │ gRPC over Unix Socket
                       ┌───────▼────────┐
                       │ NVIDIA Device  │
                       │    Plugin      │
                       └───────┬────────┘
                               │
                       ┌───────▼────────┐
                       │  NVIDIA Driver │
                       │   & Libraries  │
                       └────────────────┘
```

### 关于GPU Sharing

NVIDIA Device Plugin支持按两种方式来共享GPU资源，一种是time-slicing，一种是MPS。无论哪种方法，都是针对节点上所有GPU进行共享，不能针对单个GPU设置。

#### CUDA Time-Slicing

该选项允许负载通过时间片来共享GPU资源

```yaml
version: v1
sharing:
  timeSlicing:
    renameByDefault: <bool>
    failRequestsGreaterThanOne: <bool>
    resources:
    - name: <resource-name>
      replicas: <num-replicas>
    ...
```

在`sharing.timeSlicing.resources`中指定要共享的资源名称(e.g.,  `nvidia.com/gpu`, `nvidia.com/mig-1g.5gb`)和共享的份数。需要注意的是，这种共享方式不会隔离资源，会出现一个负载崩溃然后引发所有负载都崩溃的连锁问题。

#### CUDA MPS

该选项允许负载通过MPS技术来共享GPU资源。但要注意的是，官方文档指出当MIG启用时，不支持MPS。

```yaml
version: v1
sharing:
  mps:
    renameByDefault: <bool>
    resources:
    - name: <resource-name>
      replicas: <num-replicas>
    ...
```

### 代码走读

#### /cmd/nvidia-device-plugin

该文件夹下是主程序入口，主要进行一些插件初始化流程。

#### /internal/plugin/server.go

它的主要作用是实现 Kubernetes 定义的 **Device Plugin gRPC 接口**，充当 Kubernetes Kubelet 与底层 NVIDIA GPU 资源之间的桥梁。核心结构体`nvidiaDevicePlugin`实现了 `pluginapi.DevicePluginServer` 接口：

*   **`rm.ResourceManager`**: 负责具体的资源管理（如列出 GPU、检查健康状态）。
*   **`grpc.Server`**: 用于与 Kubelet 通信的服务器。
*   **`socket`**: Unix 域socket路径（通常在 `/var/lib/kubelet/device-plugins/`）。
*   **`deviceListStrategies`**: 设备分配策略（如环境变量、CDI、卷挂载等）。

当插件启动时，执行**`Start()`**方法：
1.  **`initialize()`** 初始化 gRPC Server 和用于停止/健康检查的 Channel。
2.  **`plugin.mps.waitForDaemon()`** 如果启用了 MPS（多进程服务），等待 MPS 守护进程就绪。
3.  **`Serve()`** 在 Unix 套接字上启动 gRPC 服务，并设置一个循环在崩溃后尝试重启。
4.  **`Register()`**: 向 Kubelet 注册，告知该plugin负责管理哪种资源（如 `nvidia.com/gpu`）和 Socket 在哪里。
5.  启动一个后台 goroutine 调用 `rm.CheckHealth`，持续监控 GPU 状态。

**`ListAndWatch()`**方法提供Kubernetes 获取设备列表的标准接口：

1.  **首次上报**: 立即向 Kubelet 发送当前所有可用设备的列表（`plugin.apiDevices()`）。
2.  **持续监听**: 进入死循环，等待 `health` Channel 的信号。
3.  **状态更新**: 一旦 `ResourceManager` 检测到某个 GPU 发生故障，该方法会接收到通知，将设备标记为 `Unhealthy` 并再次推送给 Kubelet，Kubelet 随后将不再调度新的 Pod 到该节点。

**`Allocate()`**方法提供资源分配的功能，用于封装环境配置方案，包括环境变量注入、CDI注入和设备节点挂载等。

该文件的逻辑可以概括为一个**“注册-监听-分配”**的循环：
1.  **注册**到 Kubelet。
2.  **监听** GPU 硬件健康状态并同步给 Kubernetes 调度器。
3.  当 Pod 需要 GPU 时，根据配置策略**计算**并返回容器所需的环境变量、挂载点和设备节点信息。

#### /internal/rm/rm.go

Resource Manager资源管理器的核心定义文件。

**`Devices()/Resource()`**用于获取当前资源管理器的设备列表。

**`ValidateRequest()`**用于处理共享GPU的申请：

- 检查设备ID是否存在当前节点下。
- 基于时间分片/MPS配置检查容器资源副本申请限制。

**`AddDefaultResourcesToConfig()`**负责根据当前硬件环境和插件启动参数，生成资源匹配规则。

- 对于通用GPU，将所有GPU标记为`nvidia.com/gpu`资源
- 对于支持MIG的设备，检查`MigStrategy`，`single`时将所有MIG设备视为统一的GPU资源；`mixed`时将所有MIG Profile(`mig-1g.5gb`)注册为不同的资源。

总的来说，该文件的核心作用是封装，对上(K8s)封装资源校验逻辑，确保调度请求满足插件共享策略；对下(NVML & MIG)屏蔽了复杂性，将物理硬件抽象为k8s可识别的资源配置。

#### internal/rm/nvml_device.go & nvml_manager.go

`nvml_device.go`的主要功能是实现**硬件资源的封装**，其调用底层的NVML库来获取原始硬件信息，并将其转化为K8s设备插件所需的标准化格式(UUID、设备文件路径、NUMA拓扑、显存大小等)。

`nvml_manager.go`的主要功能是通过NVML来管理和分配GPU资源。

`NewNVMLResourceManagers()`方法主要处理**资源初始化与发现**，扫描设备并按照资源名称进行分组，为每种资源创建相应的`nvmlResourceManager`实例。

`GetPreferredAllocation()`提供了容器请求GPU资源时的分配策略，一种是NVLink拓扑感知的`alignedAlloc`，一种是负载均衡的`distributedAlloc()`方法。

`CheckHealth()`方法监听设备状态，当NVML检测到硬件故障时，会通过`unhealthy channel`上报。

#### $\P$ rm组件总结

**核心抽象与数据结构**

- `rm.go`定义了主接口`ResourceManager`，抽象了底层的硬件差异，定义了一系列标准方法。
- `device.go`定义了设备模型，拓展了K8s标准的`pluginapi.Device`，增加了NVIDIA设备特有元数据，设计了AnnotatedID来使得插件支持将一个物理GPU虚拟化成多个逻辑设备。

**实现类**

- `nvml_manager.go` & `nvml_devices.go`是基于NVIDIA提供的nvml库与NVIDIA驱动互动
- `tegra_manager.go` & `tegra_devices.go`是针对嵌入式设备的实现
- `wsl_devices.go`是专门针对WSL2环境的适配逻辑

**业务逻辑**

- `allocate.go`实现了具体的设备选择策略，该文件内提供了`distributedAlloc()`策略的具体实现，`alignedAlloc()`的实现在`nvml_manager.go`下。
- `health.go`实现了对于设备的健康监测，通过监听NVML的事件来监控GPU状态，并过滤非硬件故障的报错。
- `device_map.go`负责根据插件配置，将发现的物理设备分组到不同的资源名下

该组件的关系可分阶段表述为：

1. **启动阶段**：插件根据当前环境初始化对应的 `ResourceManager`（NVML 或 Tegra）。
2. **发现阶段**：Manager 调用 `device_map.go` 结合平台特有逻辑（比如`nvml_devices.go`）生成 `Devices` 列表。
3. **运行阶段:**
   - **Kubelet 查询**：通过 `Devices()` 方法将设备报告给 Kubelet。
   - **健康监控**：`health.go` 开启后台协程，一旦发现异常，通过 `unhealthy` 通道通知插件。
   - **调度分配**：当 Pod 申请 GPU 时，`allocate.go` 计算出最优的物理/逻辑设备组合，并返回容器所需的设备节点路径。

#### $\P$ 关于CDI组件

Container Device Interface在NVIDIA Device Plugin中的作用可以总结为：

- **挂载逻辑标准化&解耦**：以前 Device Plugin 需要显式地告诉 Kubelet 每一个要挂载的文件路径；现在只需返回一个 ID（如 `nvidia.com/gpu=0`），具体的“挂载清单”全写在 CDI 的 JSON 规范文件中。
- **跨容器运行时兼容**：只要容器运行时支持 CDI 标准，NVIDIA Device Plugin 就能以统一的方式支持 Docker、containerd 和 CRI-O，而无需为每种运行时编写特殊的适配代码，消除了对于nvidia-container-runtime-hook(用于注入驱动库、设备文件等)的依赖。
- **透明性与可维护性**：配置的JSON文件持久化存储，可以查看所有设备分配的细节。

除了上述组件外，还有`vgpu`、`mig`、`imex`等针对特定技术的组件实现。
