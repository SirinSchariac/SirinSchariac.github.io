---
layout: 	post  				   
title: 		plugin manager代码走读				
subtitle:	插件管理
date:       2026-01-15 				
author:     Sirin 						
header-img: img/sunset.jpg
catalog: 	true 				
tags:						
    - Kubernetes
    - golang
---

### pkg/kubelet/pluginmanager/plugin_manager.go

#### 整体架构

`plugin_manager` 的核心是 `PluginManager` 接口及其实现类 `pluginManager`，其中核心组件包括：

- `desiredStateOfWorldPopulator`（期望状态填充器）：由 `pluginwatcher.Watcher` 管理，负责填充期望的插件状态。
- `reconciler`（调和器）：通过周期性循环将期望状态与实际状态对应，触发插件的注册与注销逻辑。
- `actualStateOfWorld`（实际状态）：数据结构，代表当前系统中已注册的插件状态。
- `desiredStateOfWorld`（期望状态）：数据结构，描述系统中期望有哪些插件被注册。

#### 核心流程

1. **初始化**：`func NewPluginManager(...)`
   - 创建 `actualStateOfWorld` 和 `desiredStateOfWorld` 数据结构，分别存储系统的实际插件状态和期望插件状态。
   - 初始化 `reconciler`：
     - `reconciler` 使用 `operationexecutor` 来处理注册和注销操作。
     - `operationexecutor` 是操作执行器，它负责调用具体的操作。
   - 初始化 `pluginwatcher.Watcher` 作为 `desiredStateOfWorldPopulator`，用来发现和更新期望的插件状态。
   - 返回一个 `pluginManager` 对象。

2. **运行**：`func (pm *pluginmanager) Run(...)` 
   1. 在运行时，优先启动 `desiredStateOfWorldPopulator`
      - 监控插件目录`sockDir`
      - 发现新的插件，并将这些插件的状态更新到 `desiredStateOfWorld`
   2. 启动 `reconciler`，周期运行一个循环：
      - 遍历 `desiredStateOfWorld` 和 `actualStateOfWorld`。
      - 如果期望状态与实际状态不一致，则通过操作生成器触发注册/注销操作，使二者一致。
   3. 注册指标监控：
      - 通过 `metrics.Register` 监控 `actualStateOfWorld` 和 `desiredStateOfWorld` 的状态。
   4. `stopCh` 信号关闭时，停止插件管理器。

3. **插件处理逻辑**：`func (pm *pluginmanager) AddHandler(...)` 
   - 用于注册新的插件处理程序，指定插件的类型`pluginType`以及处理逻辑`pluginHandler`。
   - 插件处理程序被添加至 `reconciler` 中。`reconciler` 在调和状态变化时，会调用对应的处理器。

4. **调和Reconciliation**：
   - `reconciler` 以循环的方式不断对比 `actualStateOfWorld` 和 `desiredStateOfWorld`
   - 根据状态的差异，调用相应的注册或注销操作：
     - 注册新发现的插件。
     - 注销已经不存在的插件。

#### 主要逻辑的异步实现

- **插件目录监控**`pluginwatcher`：
  - 异步发现插件的变动并填充 `desiredStateOfWorld`。
- **状态调和**`reconciler`：
  - 异步对比 `desiredStateOfWorld` 和 `actualStateOfWorld`，并执行相应的操作。

### pkg/kubelet/pluginmanager/pluginwatcher/plugin_watcher.go

实现了一个插件监听器，用于监控插件目录下的文件变化，并在变化时处理相应的注册和删除操作。

**构造函数 **：`func NewWatcher(...)`

- 用户通过调用 `NewWatcher(sockDir, desiredStateOfWorld)` 来创建一个 `Watcher` 对象。
- 这个对象包含以下关键的成员变量：
  - `path`：插件目录的路径。
  - `fs`：用于文件系统操作（默认为 Kubernetes 提供的 `DefaultFs` 实现）。
  - `desiredStateOfWorld`：Kubernetes 管理插件状态的缓存。
  - `fsWatcher`：一个文件系统观察器（`fsnotify.Watcher`），监听文件系统事件。

**启动监听器**： `func (w *Watcher) Start(ctx, stopCh)`

该方法是启动监听器的核心操作，流程如下：

1. 初始化工作目录：调用 `init(ctx)` 确保监听的目录存在，如果不存在，通过 `fs.MkdirAll` 方法递归创建目录。
2. 创建文件系统观察器：调用 `fsnotify.NewWatcher` 创建一个文件系统事件观察器并赋值给 `w.fsWatcher`。
3. 扫描现有目录结构：调用 `traversePluginDir(ctx, w.path)`，遍历插件目录，发现并处理已经存在的插件，以及将所有子目录添加到观察列表中。
4. 启动 goroutine 监听文件系统事件`fsnotify.Events`：
   - `fsnotify.Create`：处理新建插件注册（调用 `handleCreateEvent`）。
   - `fsnotify.Remove`：处理插件删除（调用 `handleDeleteEvent`）。

**事件处理**

- 文件监听到的事件由对应的事件处理函数：

  - 处理创建事件`handleCreateEvent`：

    - 获取文件信息，忽略以 `.` 开头的隐藏文件。
    - 如果是 Unix 域套接字文件，调用 `handlePluginRegistration` 完成插件注册；如果是目录则递归调用 `traversePluginDir`。

 - 处理删除事件`handleDeleteEvent`：直接将该文件的路径从 `desiredStateOfWorld` 中移除，完成撤销注册。

**辅助方法**

- `traversePluginDir`
  - 遍历监听路径中的所有子文件夹和文件。
  - 对文件夹添加文件事件监听器，对 Unix 套接字文件模拟触发 `Create` 事件处理插件注册。

- `handlePluginRegistration`
  - 核心逻辑是将插件路径或其更新的时间戳存储进 `desiredStateOfWorld` 缓存。

### pkg/kubelet/pluginmanager/reconciler/reconciler.go

该文件展示了调和器的具体实现。

reconciler结构体的内部字段包括:

- `operationExecutor`：主要负责插件的注册和反注册操作。

- `loopSleepDuration`：用于控制调和循环的频率。

- `desiredStateOfWorld`：针对插件的“理想状态”（期望存在某些插件）。
- `actualStateOfWorld`：插件的“实际注册状态”。

- `handlers`：一个线程安全的 map，存储了插件类型与其对应的处理逻辑。

- `sync.RWMutex`：用于保证并发访问时的线程安全。

reconciler中涉及的方法如下：

> `Run(stopCh <-chan struct)`

该方法的核心逻辑是运行一个调和循环，通过`wait.Until`实现定时器模式，在循环内调用reconcile方法校验并确保实际状态和理想状态一致。

> `AddHandler(pluginType string, pluginHandler cache.PluginHandler)`

该方法用于动态地为指定的插件类型添加处理逻辑，通过加锁的方式保证线程安全。

> `getHandler() map[string]cache.PluginHandler`

返回当前注册的插件类型与处理逻辑的映射表，通过读锁来确保线程安全。

> `reconcile(ctx context.Context)`

该函数实现了reconciler的核心逻辑：

1. 对不应该被注册的插件进行反注册
2. 确保所有应该被注册的插件都已经正确注册。



