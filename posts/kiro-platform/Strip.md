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

## 2. Plutus — AWS 内部支付中台

### 2.1 什么是 Plutus

**Plutus (AWSPlutusRequestRouter)** 是 AWS 内部的支付路由服务，2022 年 10 月创建，作为 AWS 服务与外部支付网关之间的统一抽象层。

Kiro 不直接调用 Stripe API，而是通过 Plutus 间接对接：

```
Kiro CPS → PlutusRequestRouterClient (Coral API) → Plutus Service → Stripe API
```

Plutus 是一个功能完整的支付平台，从其 Smithy Model 可以看到支持的能力：

| 能力 | 说明 |
|------|------|
| Checkout Session | 创建 Stripe 结账页面（新用户订阅） |
| Customer Portal | 管理已有订阅（升级/降级/取消） |
| Financing | 融资申请、账户、还款 |
| Payment Instrument | 支付工具验证 |
| Disbursement | 付款/退款 |
| Virtual Bank Account | 虚拟银行账户 |
| Product / Invoice | 产品和发票管理 |
| Connector | 可插拔的支付连接器架构（SPI） |

Kiro 只用到了其中的 **CreateSession** API。

### 2.2 Kiro 如何调用 Plutus

从 `PlutusAccessorImpl.java` 可以看到具体实现：

```java
// 注入 Plutus 客户端
private final PlutusRequestRouterClient plutusClient;

// 默认回调地址
private static final String DEFAULT_CHECKOUT_SUCCESS_URL = "https://kiro.dev/payment/success";
private static final String DEFAULT_CHECKOUT_CANCEL_URL = "https://kiro.dev/payment/error";
```

根据用户状态创建不同类型的 Session：

```java
// 场景1: 新用户订阅 → Stripe Checkout Session
CreateSessionRequest.builder()
    .clientCustomerId(userId)
    .email(email)
    .type(SessionType.CHECKOUT)                              // 结账页面
    .subscriptionType(SubscriptionType.Q_DEVELOPER_STANDALONE_PRO)  // 订阅类型
    .successUrl("https://kiro.dev/payment/success")
    .cancelUrl("https://kiro.dev/payment/error")
    .build();

// 场景2: 已有订阅用户 → Stripe Customer Portal（管理订阅）
CreateSessionRequest.builder()
    .clientCustomerId(userId)
    .email(email)
    .type(SessionType.CUSTOMER_PORTAL)                       // 管理门户
    .build();

// 场景3: 被封禁用户 → 禁用订阅管理
CreateSessionRequest.builder()
    .customerPortalType(CustomerPortalType.SUBSCRIPTION_MANAGEMENT_DISABLED)
    .build();
```

### 2.3 订阅类型映射

Plutus 内部将 Kiro 的订阅类型映射为 Stripe 的 Price/Product：

```java
// CPS 中的类型转换
switch (subscriptionType) {
    case "Q_DEVELOPER_STANDALONE_FREE":
        return SubscriptionType.Q_DEVELOPER_STANDALONE_FREE;
    case "Q_DEVELOPER_STANDALONE_PRO":
        return SubscriptionType.Q_DEVELOPER_STANDALONE_PRO;       // $20/月
    case "Q_DEVELOPER_STANDALONE_PRO_PLUS":
        return SubscriptionType.Q_DEVELOPER_STANDALONE_PRO_PLUS;  // $40/月
    case "Q_DEVELOPER_STANDALONE_POWER":
        return SubscriptionType.Q_DEVELOPER_STANDALONE_POWER;     // $200/月
}
```

### 2.4 Plutus 在支付链路中的位置

```
用户点击 "升级到 Pro"
    │
    ▼
Kiro Client → CPS.CreateSubscriptionToken(provider=STRIPE, type=PRO)
    │
    ├─ 1. 验证用户身份 (Builder ID / Kiro Social)
    ├─ 2. Zorn DP → 获取当前订阅状态 (hasSubscription? isBlocked?)
    ├─ 3. IdentityStore → 获取用户 email
    │
    ▼
PlutusAccessor.createSessionForExistingOrNewUser()
    │
    ├─ 无订阅 → SessionType.CHECKOUT → Stripe Checkout 页面
    ├─ 有订阅 → SessionType.CUSTOMER_PORTAL → Stripe 管理门户
    └─ 被封禁 → SUBSCRIPTION_MANAGEMENT_DISABLED
    │
    ▼
返回 Session URL → 用户浏览器跳转到 Stripe
    │
    ▼
用户完成支付 → Stripe Webhook → Zorn 更新订阅状态
```

## 3. 付费流程 (完整链路)

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
    │   │      → Plutus → Stripe 创建 Checkout/Portal Session
    │   │      → 传入: userId, email, subscriptionType, hasSubscription, isUserBlocked
    │   └─ 5. 返回 Stripe Session URL (encodedVerificationUrl)
    │
    └─ Provider = Account Linking (备用路径，当前 blocked)
        └─ 仅支持 Builder ID 用户查询订阅状态 (statusOnly=true)
```

关键依赖：

- **Plutus** (`PlutusRequestRouterClient`) → AWS 内部支付中台，对接 Stripe
- **Zorn DataPlane** (`ZornDataPlaneAccessor`) → 订阅状态管理和用量追踪
- **IdentityStore** (`IdentityStoreAccessor`) → 用户身份和 email 获取

## 4. 用量追踪与限制

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

## 5. 超额 (Overage) 机制

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

## 6. 用户类型与权限控制

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

## 7. Feature Flag 控制

```java
// 关键 Feature Flags:
SupportedFeature.KIRO_PRICING_MODEL_ENABLED  // Kiro 付费模型开关
SupportedFeature.KIRO_ENTERPRISE_ENABLED     // Kiro 企业版开关
SupportedFeature.USAGE_LIMITS_API_ENABLED    // 用量限制 API 开关
```

## 8. KiroAuthService 在付费中的角色

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
    
Plutus (AWS 内部支付中台)
    ├─ CreateSession → Stripe Checkout / Customer Portal
    ├─ 订阅类型映射 (Kiro Plan → Stripe Price)
    └─ 封禁用户处理
        │
        ▼
    Stripe (外部支付网关)
        └─ 实际扣款处理
```
