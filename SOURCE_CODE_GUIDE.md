# Volcano 源码阅读指南

## 目录

- [项目简介](#项目简介)
- [整体架构](#整体架构)
- [代码结构](#代码结构)
- [核心概念](#核心概念)
- [研读路线](#研读路线)
- [关键模块详解](#关键模块详解)

---

## 项目简介

**Volcano** 是一个 Kubernetes 原生的批处理调度系统，扩展并增强了标准 kube-scheduler 的能力。

- **适用场景**: AI/ML/DL 训练、生物信息学、大数据应用
- **支持框架**: Spark, Flink, Ray, TensorFlow, PyTorch, Kubeflow, MPI, Argo 等
- **所属组织**: CNCF 孵化项目

---

## 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        Volcano                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Scheduler     │    │  Controllers │    │    Agent     │ │
│  │                 │    │              │    │              │ │
│  │ Schedule kernel │    │  CRD contrl  │    │  node agent  │ │
│  └─────────────────┘    └──────────────┘    └──────────────┘ │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                    CRD                                       │
│  ┌────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │   Job      │   │  Queue      │   │  PodGroup   │          │
│  │  batch job │   │ Schedule Que│   │  Pod Group  │          │
│  └────────────┘   └─────────────┘   └─────────────┘          │
└──────────────────────────────────────────────────────────────┘
```


## 代码结构

```
volcano/
├── cmd/                        # 各组件入口
│   ├── scheduler/             # 调度器 ← 核心
│   ├── controller-manager/    # 控制器
│   ├── agent/                 # 节点 agent
│   └── cli/                   # 命令行工具
│
├── pkg/                        # 核心业务逻辑
│   ├── scheduler/             # 调度器核心代码
│   │   ├── scheduler.go       # Scheduler 主结构
│   │   ├── framework/         # 调度框架接口 ← 关键
│   │   ├── plugins/           # 各种调度插件
│   │   ├── actions/           # 调度动作
│   │   ├── cache/             # 调度缓存
│   │   └── api/               # 调度 API
│   │
│   ├── controllers/           # CRD 控制器
│   └── webhooks/              # 准入控制
│
├── staging/src/volcano.sh/apis/  # CRD 定义 (Job, Queue, PodGroup)
├── config/crd/                 # CRD YAML
└── installer/                  # 安装配置
```

---

## 核心概念

### 1. Session (调度会话)

一次完整的调度周期，包含当前集群状态、待调度任务等。

```go
type Session struct {
    Cache    // 集群状态缓存
    Jobs     // 待调度任务
    Queues   // 调度队列
    Nodes    // 可用节点
    Plugins  // 调度插件
}
```

### 2. Action (调度动作)

定义调度流程中的**某个阶段**做什么。

```go
type Action interface {
    Name() string
    Initialize()
    Execute(ssn *Session)
    UnInitialize()
}
```

**常见 Actions**:

| Action     | 作用        |
|------------|-----------|
| `enqueue`  | 将任务加入队列   |
| `reclaim`  | 跨队列回收资源   |
| `allocate` | 分配资源 (核心) |
| `backfill` | 填充碎片资源    |
| `preempt`  | 同队列抢占     |
| `shuffle`  | 任务重排序     |

### 3. Plugin (调度插件)

提供**具体调度策略**。

```go
type Plugin interface {
    Name() string
    OnSessionOpen(ssn *Session)
    OnSessionClose(ssn *Session)
}
```

**常见 Plugins**:

| Plugin          | 作用                         |
|-----------------|----------------------------|
| `gang`          | Gang Scheduling            |
| `drf`           | Dominant Resource Fairness |
| `predicates`    | 节点过滤                       |
| `priority`      | 优先级                        |
| `nodeorder`     | 节点打分排序                     |
| `capacity`      | 容量调度                       |
| `task-topology` | 任务拓扑                       |

---

## 研读路线

### 阶段 1: 理解核心接口

```
1. pkg/scheduler/framework/interface.go
   └── 理解 Action、Plugin、Session 的抽象设计

2. staging/src/volcano.sh/apis/
   └── 了解 Job、Queue、PodGroup 等 CRD 定义
```

### 阶段 2: 调度器主流程

```
1. pkg/scheduler/scheduler.go
   └── Scheduler 结构、Run()、runOnce()

2. pkg/scheduler/framework/session.go
   └── Session 生命周期管理

3. pkg/scheduler/cache/
   └── 调度缓存实现
```

### 阶段 3: Actions 调度流程

```
pkg/scheduler/actions/
├── enqueue/      任务入队
├── reclaim/      跨队列回收
├── allocate/     资源分配 ← 重点
├── backfill/     填充碎片
└── preempt/      抢占
```

### 阶段 4: Plugins 调度策略

```
pkg/scheduler/plugins/
├── gang/         Gang 调度
├── drf/          DRF 公平调度
├── predicates/   节点过滤
├── nodeorder/    节点打分
├── priority/     优先级
└── capacity/     容量调度
```

### 阶段 5: Controllers 控制器

```
pkg/controllers/
├── job/          Job 控制器
├── queue/        Queue 控制器
└── podgroup/     PodGroup 控制器
```

---

## 关键模块详解

### Scheduler 主流程

**文件**: `pkg/scheduler/scheduler.go`

```go
// 启动流程
Run(stopCh)
    ├── loadSchedulerConf()      // 加载配置
    ├── watchSchedulerConf()     // 监听配置热更新
    ├── cache.Run()              // 启动缓存
    └── wait.Until(runOnce)      // 周期性调度

// 单次调度周期
runOnce()
    ├── 获取 actions/plugins 配置
    ├── OpenSession()            // 创建调度会话
    ├── for action in actions:   // 依次执行 Action
    │   └── action.Execute(ssn)
    └── CloseSession()           // 关闭会话
```

### Actions 调度流水线

```
          ┌─────────────┐
          │   enqueue   │  任务入队
          └──────┬──────┘
                 │
          ┌──────▼──────┐
          │   reclaim   │  跨队列回收（高优抢低优）
          └──────┬──────┘
                 │
          ┌──────▼──────┐
          │  allocate   │  ◄─── 核心分配逻辑
          └──────┬──────┘
                 │
          ┌──────▼──────┐
          │  backfill   │  填充碎片
          └──────┬──────┘
                 │
          ┌──────▼──────┐
          │   preempt   │  同队列抢占
          └─────────────┘
```

### Allocate Action 核心逻辑

**文件**: `pkg/scheduler/actions/allocate/allocate.go`

```
allocate.Execute(ssn)
    ├── 按优先级排序队列
    ├── for each queue:
    │   ├── 按优先级排序任务
    │   └── for each job:
    │       └── for each task:
    │           ├── predicates 过滤节点
    │           ├── priorities 对节点打分
    │           └── 选择最优节点并绑定
```

### Plugin 扩展机制

插件通过 Session 在调度周期的不同阶段介入：

```
OpenSession()
    └── for plugin in plugins:
        └── plugin.OnSessionOpen(ssn)

[Actions 执行...]

CloseSession()
    └── for plugin in plugins:
        └── plugin.OnSessionClose(ssn)
```

---

## 调试技巧

### 查看调度配置

```bash
kubectl get configmap -n volcano-system volcano-scheduler-configmap -o yaml
```

### 查看调度日志

```bash
kubectl logs -n volcano-system deployment/volcano-scheduler
```

### 启用详细日志

```bash
# 修改 scheduler 配置，设置 -v=4 或更高
```

---

## 扩展阅读

- [官方文档](https://volcano.sh/)
- [设计文档](./docs/design/)
- [使用指南](./docs/getting-started/getting-started.md)
- [KubeCon 演讲视频](README.md#talks)

---

## 贡献指南

如发现文档问题或需要补充，欢迎提交 PR 或 Issue。
