## 三个服务的角色定位

1. KiroAuthService (认证服务)
- 描述: "Coral Service on Fargate"
- Owner: kiro-core@amazon.com, manager: raokar
- 部署: us-east-1 (beta/gamma/prod)
- 端点格式: https://{domain}.us-east-1.auth.desktop.kiro.dev
- 关键包: KiroAuthService, KiroAuthWorkers, KiroAuthZebra (Zebra 用于异步任务)
- 部署路径: beta-onepod → beta-fleet → gamma-onepod → gamma-fleet → prod-onepod → prod-fleet

2. AWSVectorConsolasRuntimeService (RTS) (运行时服务)
- 描述: "A CDK Managed pipeline for AWSVectorConsolasRuntimeServiceCDK"
- Owner: consolas-eng@amazon.com (原 CodeWhisperer/Q Developer 团队), manager: satpatia
- RIP Service: codewhisperer+qdev-rts
- 多区域部署: PDX, IAD, FRA + GovCloud (PDT, OSU) + ADC (DCA, LCK, ALE, LTW)
- 关键测试覆盖的付费相关功能:
  - **Usage Throttling** 测试 (AWSID_USAGE_THROTTLED, SSO_USAGE_THROTTLED) — 用量限制/计费
  - **Usage Limits** 测试 — 使用额度限制
  - **Control Plane Integration** 测试 — 与 CPS 的 Pro Tier 集成 (bearerTokenUserName 包含 pro-tier-integ)
  - **SSO / AWSID / SSO_WITH_CMCMK** 多种认证方式
  - **Kiro Enterprise / Kiro Social / Kiro Fraud** 集成测试 — Kiro 特有的企业版、社交和反欺诈功能
  - **Custom Inference / Customization** — 自定义模型推理

3. AWSVectorConsolasControlPlaneService (CPS) (控制面服务)
- 描述: "A CDK Managed pipeline for AWSVectorConsolasControlPlaneService"
- RIP Service: codewhisperer+qdev-cps
- Owner: consolas-eng@amazon.com, manager: raokar
- 关键测试覆盖的付费相关功能:
  - **Profile 管理** (profile functional tests) — 用户 profile/订阅管理
  - **Grants 管理** (grants functional tests) — 授权/许可管理
  - **Plugin 管理** (plugin integration tests, GQ Plugin, GitHub Plugin) — 插件/扩展管理
  - **Tagris** 测试 — AWS 资源标签合规
  - **Customer Managed Key (CMK)** 支持 — 客户自管密钥
  - **Profile Migration Security** 测试 — 付费 profile 迁移安全
  - **Pro Tier Integration** — 专业版集成 (bearerTokenUserName 包含 pro-tier-integ)

## Kiro 付费支持的架构流程

Kiro Client (Desktop/CLI/IDE)
        │
        ▼
  KiroAuthService ──── 认证 & 身份验证
  (auth.desktop.kiro.dev)   支持 SSO / AWSID / Bearer Token
        │
        ▼
  ┌─────┴──────┐
  ▼            ▼
 CPS          RTS
 (控制面)     (运行时)
  │            │
  │ Profile    │ Usage Throttling
  │ Grants     │ Usage Limits
  │ Plugin     │ Pro Tier 检查
  │ CMK        │ Enterprise/Social/Fraud
  │ Tagris     │ Custom Inference
  └─────┬──────┘
        ▼
   付费功能执行


具体付费支持机制:

1. 认证层 (KiroAuthService): 处理用户身份认证，支持 SSO 和 AWSID 两种 bearer token 类型，是付费验证的入口

2. 订阅管理 (CPS): 通过 Profile API 管理用户的订阅状态（Free/Pro tier），Grants API 管理授权许可，支持 CMK 加密

3. 用量控制 (RTS):
   - Usage Throttling: 对达到用量限制的用户进行限流
   - Usage Limits: 检查和执行使用额度
   - Pro Tier 集成: 根据 CPS 返回的订阅级别决定功能可用性

4. Kiro 特有功能 (RTS):
   - Enterprise 测试: 企业版付费功能验证
   - Fraud 检测: 防止付费欺诈
   - Social 功能: 社交/协作付费功能
