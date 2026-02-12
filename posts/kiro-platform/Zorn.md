# Zorn — Kiro 订阅与用量计量后端

Zorn 是 AWS 内部的**用户订阅管理和用量计量服务**，是 Kiro 付费体系的核心后端。Kiro 的 CPS 和 RTS 都依赖 Zorn 来完成订阅状态查询、用量聚合和超额追踪。

## 服务概览

| 服务 | Pipeline | 描述 | 类型 |
|------|----------|------|------|
| AWSZornControlPlaneService | [Pipeline](https://pipelines.amazon.dev/pipelines-wip/AWSZornControlPlaneService) | 订阅管理控制面 | Coral Service on Fargate |
| AWSZornDataPlaneService | [Pipeline](https://pipelines.amazon.dev/pipelines-wip/AWSZornDataPlaneService) | 用量数据查询面 | Coral Service on Fargate |
| AWSZornMetering | [Pipeline](https://pipelines.amazon.dev/pipelines-wip/AWSZornMetering) | 用量计量 Lambda | Java Lambda (CDK) |

- **Owner**: iamsrk (所有三个服务)
- **共享 Version Set**: `AWSZorn/development`（三个服务 merge 同一个 VS）
- **监控**: 每个区域都有 `AWSZorn-AggregateHealthMonitor-{stage}-{region}` 告警

## 各服务职责

### 1. Zorn Control Plane Service（订阅管理）

管理用户的订阅生命周期：

- **创建/更新/删除订阅** — 当用户通过 Stripe 付费后，CPS 调用 Zorn CP 创建订阅记录
- **订阅状态查询** — 查询用户当前是 Free/Pro/Pro+/Power 哪个层级
- **订阅激活信号** — `NotifyApplicationUsage` 通知 Zorn 用户正在使用服务
- **超额控制** — 管理用户的 overage 开关状态

Kiro CPS 中的调用方：
- `ZornControlPlaneAccessor` — 订阅管理操作
- `ZornOverageAccessor` — 超额状态管理

### 2. Zorn Data Plane Service（用量查询）

提供实时和聚合的用量数据：

- **`GetUserSubscriptionStatus`** — 查询用户订阅状态（是否有订阅、是否被封禁）
- **`GetAggregatedUsage`** — 获取用户的聚合用量数据，包含：
  - `usageBreakdownList` — 按资源类型（CREDIT 等）的用量明细
  - `subscriptionInfo` — 订阅信息（类型、状态）
  - `overageConfiguration` — 超额配置
  - `totalUsage` — 总用量和超额上限

Kiro 中间层的调用方：
- CPS: `ZornDataPlaneAccessor.getUserSubscriptionStatus()` — 创建订阅 token 时检查状态
- CPS: `ZornDataPlaneAccessor.getAggregatedUsageResponseWithAllUsageBreakdowns()` — 用量查询
- RTS: `ZornAggregatedUsageGetter` — 带缓存/不带缓存两种模式获取聚合用量
- RTS: `ZornSubscriptionStatusGetter` — 获取订阅状态用于限流决策

### 3. Zorn Metering（用量计量）

Lambda 函数，负责实时记录用量事件：

- **用量事件采集** — 每次 API 调用（代码补全、Chat、Code Scan 等）产生的 credit 消耗
- **用量聚合** — 将原始事件聚合为按用户、按资源类型的用量统计
- **计费数据生成** — 为 Stripe 超额计费提供数据源

RTS 中的用量记录链路：
```
用户调用 API → RTS Interceptor 记录用量 → Zorn Metering 采集 → Zorn DP 聚合查询
```

## 部署覆盖

三个服务都是全球部署，覆盖范围一致：

| 分区 | 区域 |
|------|------|
| **Commercial** | us-east-1, us-east-2, us-west-1, us-west-2, eu-central-1, eu-central-2, eu-west-1, eu-west-2, eu-west-3, eu-north-1, eu-south-1, ap-northeast-1, ap-northeast-2, ap-northeast-3, ap-southeast-1, ap-southeast-2, ap-southeast-3, ap-south-1, ca-central-1, sa-east-1, af-south-1, me-central-1 |
| **GovCloud** | us-gov-east-1, us-gov-west-1 |
| **ADC (ISO)** | us-iso-east-1, us-isob-east-1, us-isof-east-1, us-isof-south-1 |

部署路径（以 CP/DP 为例）：
```
alpha-us-west-2 → gamma-us-west-2 (onebox→fleet) → gamma-us-east-1 (onebox→fleet)
    → prod-us-east-2 (onebox→fleet) → prod-us-west-2 → prod-us-west-1 → prod-us-east-1
    → prod-eu-* / prod-ap-* / prod-sa-* / prod-af-*
    → prod-us-gov-* → prod-us-iso-* / prod-us-isob-* / prod-us-isof-*
```

## 在 Kiro 付费中的位置

```
Kiro Client
    │
    ▼
KiroAuthService (认证)
    │
    ▼
┌────────────────────────────────────┐
│  CPS (控制面)     RTS (运行时)      │
│  │                │                │
│  │ Profile        │ Usage          │
│  │ Subscription   │ Interceptors   │
│  │ Grants         │ Throttling     │
│  └───────┬────────┘                │
│          │                         │
│          ▼                         │
│  ┌─── Zorn ───────────────────┐    │
│  │                            │    │
│  │  Control Plane Service     │    │
│  │  ├─ 订阅创建/更新/删除      │    │
│  │  └─ 超额开关管理           │    │
│  │                            │    │
│  │  Data Plane Service        │    │
│  │  ├─ GetSubscriptionStatus  │    │
│  │  └─ GetAggregatedUsage    │    │
│  │                            │    │
│  │  Metering Lambda           │    │
│  │  ├─ 用量事件采集           │    │
│  │  └─ Credit 消耗记录        │    │
│  │                            │    │
│  └────────────────────────────┘    │
│          │                         │
│          ▼                         │
│     Plutus → Stripe (支付)         │
└────────────────────────────────────┘
```

## 关键数据流

### 订阅创建
```
用户选择计划 → CPS.CreateSubscriptionToken
    → Zorn DP.GetUserSubscriptionStatus (检查当前状态)
    → Plutus.createSession (创建 Stripe 会话)
    → 用户完成支付
    → Zorn CP 创建订阅记录
```

### 用量检查（每次 API 调用）
```
RTS 收到请求 → Usage Interceptor 触发
    → ZornAggregatedUsageGetter (带缓存) 获取当前用量
    → 判断是否超额 → 允许/限流/拒绝
    → Zorn Metering 记录本次消耗
```

### 用量展示（IDE 中查看）
```
IDE 请求 GetUsageLimits (isEmailRequired=true)
    → ZornAggregatedUsageGetter (不带缓存，实时数据)
    → 返回 usageBreakdowns + subscriptionInfo + overageConfig
    → IDE 展示用量仪表盘
```
