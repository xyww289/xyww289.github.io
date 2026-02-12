# Kiro 付费体系架构

## 1. 订阅计划 (4个层级)

从 `KiroListAvailableSubscriptionsActivity.java` 可以看到：

| 计划 | 价格 | Credits | 对应 Q Developer 类型 |
|------|------|---------|----------------------|
| KIRO_FREE | $0/月 | 50 credits | `Q_DEVELOPER_STANDALONE_FREE` |
| KIRO_PRO | $20/月 | 1,000 credits | `Q_DEVELOPER_STANDALONE_PRO` |
| KIRO_PRO_PLUS | $40/月 | 2,000 credits | `Q_DEVELOPER_STANDALONE_PRO_PLUS` |
| KIRO_POWER | $200/月 | 10,000 credits | `Q_DEVELOPER_STANDALONE_POWER` |

所有付费计划都支持 **Pay-per-use overage**（超额按量付费）。

## 2. 付费流程 (Stripe 集成)

从 `KiroCreateSubscriptionTokenActivity.java` 可以看到完整的订阅创建流程：

```
用户发起订阅请求
    │
    ▼
KiroAuthService 验证 Bearer Token
    │ (只允许 BUILDER_ID 和 KIRO_SOCIAL 用户)
    │
    ▼
CPS: KiroCreateSubscriptionTokenActivity
    │
    ├─ Provider = STRIPE (主要路径)
    │   │
    │   ├─ 1. 验证用户身份 (Builder ID / Kiro Social)
    │   ├─ 2. 从 Zorn DataPlane 获取当前订阅状态
    │   ├─ 3. 从 IdentityStore 获取用户 email
    │   ├─ 4. 调用 PlutusAccessor.createSessionForExistingOrNewUser()
    │   │      → 创建 Stripe Checkout Session
    │   │      → 传入: userId, email, subscriptionType, hasSubscription, isUserBlocked
    │   └─ 5. 返回 Stripe Session URL (encodedVerificationUrl)
    │
    └─ Provider = Account Linking (备用路径)
        └─ 仅支持 Builder ID 用户查询订阅状态
```

关键依赖：

- **Plutus** (AWS 内部支付服务) → 对接 Stripe 支付网关
- **Zorn DataPlane** → 订阅状态管理和用量追踪
- **IdentityStore** → 用户身份和 email 获取

## 3. 用量追踪与限制

从 RTS 的 `GetUsageLimitsActivity.java` 和 CPS 的 `KiroUsageLimitsActivity.java`：

```
用户使用 Kiro 功能 (Chat/Code/Agent)
    │
    ▼
RTS: Usage Interceptors (拦截器层)
    │ ├─ UsageInterceptorV2 → 记录每次调用的用量
    │ ├─ RecommendationThrottlingInterceptor → 代码推荐限流
    │ ├─ ConversationalThrottlingInterceptor → 对话限流
    │ └─ CodeScanThrottlingInterceptor → 代码扫描限流
    │
    ▼
Zorn Service (用量计量后端)
    │ ├─ ZornAggregatedUsageGetter → 获取聚合用量
    │ ├─ ZornSubscriptionStatusGetter → 获取订阅状态
    │ └─ ZornControlPlaneAccessor → 订阅管理
    │
    ▼
返回给客户端:
    ├─ usageBreakdownList (按 ResourceType.CREDIT 分类)
    ├─ subscriptionInfo (订阅信息)
    ├─ overageConfiguration (超额配置)
    ├─ nextDateReset (下次重置日期)
    └─ totalUsage (总用量 + overageLimit)
```

## 4. 超额 (Overage) 机制

```java
// RTS GetUsageLimitsActivity 中的超额计算
if (usageRecordV3.getTotalUsage().getOverageLimit() != null
        && usageBreakdown.getOverageRate() != null
        && usageBreakdown.getOverageRate() > 0) {
    overageCap = totalUsage.getOverageLimit() / usageBreakdown.getOverageRate();
}
```

- 只有 **Kiro Social** 和 **Builder ID** 用户以及 **Kiro Enterprise** 用户可以看到 `overageConfiguration`
- 超额上限 = `overageLimit / overageRate`

## 5. 用户类型与权限控制

```
用户类型判断:
├─ BUILDER_ID    → 个人用户 (Builder ID 登录)
├─ KIRO_SOCIAL   → 社交登录用户 (GitHub 等)
├─ EXTERNAL_IDP  → 企业外部 IdP 用户
└─ 其他           → IAM Identity Center 企业用户

访问控制:
├─ Kiro Social / Builder ID     → 可访问订阅 API、用量 API
├─ Q Developer Pro 订阅用户     → 被拒绝访问 Kiro (互斥)
├─ Enterprise 用户              → 通过 KIRO_ENTERPRISE_ENABLED feature flag 控制
└─ Amazon Internal 用户         → 特殊处理 (amazonInternalProfileOverride)
```

## 6. Feature Flag 控制

```java
// 关键 Feature Flags:
SupportedFeature.KIRO_PRICING_MODEL_ENABLED  // Kiro 付费模型开关
SupportedFeature.KIRO_ENTERPRISE_ENABLED     // Kiro 企业版开关
SupportedFeature.USAGE_LIMITS_API_ENABLED    // 用量限制 API 开关
```

## 7. KiroAuthService 在付费中的角色

从 `RTSService.kt` 可以看到：

- KiroAuthService 通过调用 RTS 的 `GetUsageLimits` API 获取用户信息
- 使用 Bearer Token 认证
- 获取 `userId`、`identityStoreId`、`email` 等信息
- 这些信息用于后续的订阅和计费流程

## 总结流程图

```
Kiro Client
    │
    ▼
KiroAuthService ─── 认证 (Cognito/SSO/GitHub)
    │                 ↕ RTSService.getIdCUserInfo()
    ▼
CPS (控制面)                         RTS (运行时)
├─ ListAvailableSubscriptions        ├─ GetUsageLimits (用量查询)
│  → 返回4个计划定价                  │  → Zorn 聚合用量
├─ CreateSubscriptionToken           ├─ UpdateUsageLimits (限额调整)
│  → Plutus → Stripe 支付            ├─ Usage Interceptors (实时计量)
├─ GetUsageLimits                    │  → 每次 API 调用记录 credit 消耗
│  → Zorn 用量数据                   └─ Subscription Validation
└─ Profile/Grants 管理                   → 验证订阅有效性
    │                                    → 限流/拒绝超额请求
    ▼
Zorn (AWS 内部订阅计量服务)
    ├─ 订阅状态管理
    ├─ 用量聚合计算
    └─ 超额追踪

Plutus → Stripe
    └─ 实际支付处理
```
