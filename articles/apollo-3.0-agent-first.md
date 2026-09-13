# Apollo 3.0：迈向 Agent First 的配置管理

大模型正在从“生成内容”走向“执行任务”。Agent 已经开始参与编码、测试、运维和软件交付流程。当 Agent 真正进入这些工作流后，读取配置、修改配置、发布和回滚，也会逐渐成为自然需求。

但配置中心与普通工具不同。它既承载着生产环境的运行控制，也可能包含敏感配置。面向 Agent 开放配置管理能力，不能只是增加几个 API，或者把一个高权限 Token 交给 Agent。

我们对 Agent First 的理解是：

> Agent First 不是在 Apollo 中增加一个 AI 对话框，而是让 Agent 能够理解 Apollo、获得清晰的授权、安全地执行操作，并且可以验证它是否做对、是否越界。

围绕这一方向，社区在 [Agent-first design proposal #5573](https://github.com/apolloconfig/apollo/issues/5573) 中展开了持续讨论，并逐步形成了一个渐进的演进路径：

> **OpenAPI 让 Agent 可操作，Token 让 Agent 可授权，CLI 让 Agent 可使用，Evals 让 Agent 可度量。**

Apollo 3.0 是这条道路上的第一个重要里程碑。

## OpenAPI 让 Agent 可操作

要让 Agent 操作一个系统，首先需要一套稳定、结构化、机器可理解的控制面。

过去，Apollo Portal 的操作主要依赖面向前端的内部接口，OpenAPI 更多是提供给外部集成使用。两套接口长期并存，不仅容易产生能力差异，也会增加自动化工具理解和调用 Apollo 的成本。

在 Apollo 3.0 中，Portal 的配置项、Namespace、发布与灰度、实例、权限、AccessKey、导入导出等核心管理流程已经全面迁移到 OpenAPI。Portal 自身与外部系统开始使用同一套接口契约，OpenAPI 不再只是附加的集成入口，而是 Apollo 管理面的基础。

相关工作包括 [#5608](https://github.com/apolloconfig/apollo/pull/5608)、[#5610](https://github.com/apolloconfig/apollo/pull/5610)、[#5612](https://github.com/apolloconfig/apollo/pull/5612)、[#5616](https://github.com/apolloconfig/apollo/pull/5616)、[#5617](https://github.com/apolloconfig/apollo/pull/5617) 和 [#5618](https://github.com/apolloconfig/apollo/pull/5618)。

对于 Agent 而言，这意味着它不需要理解 Portal 页面，也不需要依赖不稳定的内部接口，而是可以通过明确的资源模型和 API 契约完成配置管理任务。

## Token 让 Agent 可授权

能够调用 API，并不意味着可以把权限安全地交给 Agent。

传统的 Consumer Token 更适合代表一个应用或内部平台，往往具有相对固定的身份和较大的权限范围。对于由用户发起的 AI 辅助操作，我们更需要一种能够表达“谁委托了 Agent，以及这次委托允许做什么”的授权方式。

Apollo 3.0 新增了面向用户、Agent 和个人自动化场景的 [User Access Token](https://github.com/apolloconfig/apollo/pull/5632)。

用户可以在 Portal 中创建代表自身身份的 Token，并从多个维度限制其能力：

- 允许执行的操作；
- 可以访问的 AppId、环境、集群和 Namespace；
- Token 的有效期和速率限制；
- Token 的撤销、轮换与使用审计。

User Access Token 不会复制一份用户角色。Apollo 会在每次请求时，重新计算用户当前权限与 Token Scope 的交集：

```text
有效权限 = 用户当前 RBAC 权限 ∩ Token Scope
```

因此，当用户权限被收回、账号被禁用、Token 过期或被撤销时，相关访问也会立即失效或降权。Token 明文只在创建或轮换时展示一次，服务端仅保存哈希和前缀。

Apollo 还提供了机器可读的能力发现接口。Agent 可以先查询当前身份、资源范围以及允许调用的 OpenAPI Action，再决定后续操作，而不是通过试错来猜测权限。

这使 Agent 获得的不再是一把长期、宽泛的共享钥匙，而是一份由用户委托、范围明确、可撤销且可审计的访问能力。

## CLI 让 Agent 可使用

有了统一的 OpenAPI 和用户委托授权后，还需要一个稳定、易调用的工具入口。

围绕 Apollo 3.0，社区推出了独立的官方 [apollo-cli](https://github.com/apolloconfig/apollo-cli)。它基于 Apollo Portal OpenAPI 构建，为人、脚本、CI 以及 Codex、Claude Code 等 AI Coding Agent 提供一致的命令行界面。

当前 CLI 已覆盖应用、环境、Namespace、配置、发布和回滚等常见工作流，并针对 Agent 和自动化场景提供了：

- 稳定的 JSON 输出和结构化错误；
- User Access Token 与 Consumer Token 支持；
- 系统凭证存储与多环境 Profile；
- Token、Authorization Header 等敏感信息的默认脱敏；
- 变更前的目标与操作计划展示；
- 修改、发布、回滚等操作的确认保护；
- 基于 OpenAPI 的通用 API 调用入口。

CLI 并不仅是对 REST API 的简单封装。它把 Apollo 的接口契约转换成了一套更稳定、更容易被 Agent 理解和执行的工具语义，同时为敏感信息和变更操作提供了更加保守的默认行为。

## Evals 让 Agent 可度量

一个 Agent 能够成功调用命令，并不等于它正确完成了任务。

例如，Agent 可能修改了正确的配置，却发布到了错误的环境；也可能完成了目标操作，同时误改了另一个相似的应用。仅检查命令是否执行成功，无法判断这些问题。

因此，围绕 Apollo 3.0，社区进一步建立了独立的 [apollo-evals](https://github.com/apolloconfig/apollo-evals) 评测体系，在真实、隔离的 Apollo 环境中评估不同 Agent、模型和工具组合完成配置管理任务的能力。

每个评测任务都会同时验证三个维度：

- **Outcome**：最终配置和发布状态是否正确；
- **Interaction**：Agent 是否真正使用了指定的 Apollo 产品能力；
- **Boundary**：其他应用、Namespace 和非目标资源是否保持不变。

首批场景已经覆盖配置修改与发布、版本回滚、Namespace 创建、跨环境配置同步、Token 能力范围发现，以及 Apollo Java Client 使用等真实任务。

Evals 让社区不仅能够展示“Agent 可以操作 Apollo”，还可以持续、客观地回答：

- 哪些任务 Agent 已经能够可靠完成？
- 哪些安全边界容易被突破？
- 不同模型和 Agent Harness 的表现如何？
- Apollo 的一次产品改进，是否真的提升了任务成功率和安全性？

这也为 Agent First 的持续演进建立了一个可重复、可验证的反馈循环。

## 从一次配置变更看完整链路

假设用户希望 Agent 修改某个应用 DEV 环境下的配置并发布。

用户可以创建一个仅允许访问目标 App、DEV 环境和指定 Namespace，同时只具备配置读取、修改与发布能力的 User Access Token。

Agent 首先查询 Token 的能力范围，然后通过 Apollo CLI 查看现有配置和差异，执行修改并创建发布。CLI 会展示经过脱敏的操作计划，Portal 则在每次请求时校验用户当前权限与 Token Scope，并记录 Token 使用情况。

在评测环境中，Apollo Evals 还可以进一步验证：目标配置是否已经生效、发布是否成功，以及其他相似应用和非目标配置是否保持不变。

这条链路体现了 Apollo 当前对 Agent First 的理解：不仅要让 Agent“能够完成操作”，还要让操作有明确的身份、范围、安全边界和验证结果。

## 这只是起点

Apollo 3.0 已经建立了面向 Agent 的接口、授权、工具和评测基础，但还没有实现完整的 Agent 原生治理模型。

当前的 User Access Token 本质上是建立在用户 RBAC 权限之上的细粒度委托授权。它仍然是一种通用 Bearer Credential，而不是面向某个具体任务的短生命周期会话凭证。

下一步可能是在现有 RBAC 和 Token Scope 之上，引入面向任务的访问控制与 Agent Session：

```text
有效权限
  = 发起用户当前的 RBAC 权限
  ∩ Session / Task Scope
  ∩ Action Policy
  ∩ Approval State
```

未来需要继续探索的问题包括：

- 如何将 Agent 授权绑定到具体任务和有限生命周期；
- 哪些操作可以直接执行，哪些只能生成草稿或必须经过审批；
- 如何区分配置元数据、脱敏值和原始敏感值的读取权限；
- 如何记录用户、Agent Session、操作提案、审批和最终执行之间的完整审计链路；
- 如何让 CLI、未来的 MCP 以及其他 Agent 工具共享同一套安全治理边界。

相关设计正在 [Design user-sponsored agent sessions for Apollo #5627](https://github.com/apolloconfig/apollo/issues/5627) 中持续讨论。

## Apollo 3.0 的其他演进

除了 Agent First 方向，Apollo 3.0 还完成了一系列服务端和配置管理能力升级。

### 服务端技术基线升级

Apollo 服务端升级到 Java 17、Spring Boot 4.1.1、Spring Cloud 2025.1.3 和 Spring Framework 7，并完成了相关组件和服务发现实现的适配。

相关工作见 [#5585](https://github.com/apolloconfig/apollo/pull/5585) 和 [#5671](https://github.com/apolloconfig/apollo/pull/5671)。

### 更轻量的默认部署方式

Apollo Config Service 和 Admin Service 的官方安装包默认启用 `database-discovery`，直接利用 ApolloConfigDB 完成服务注册与发现，降低默认部署和运维的组件复杂度。

原有 Eureka 模式仍然受到支持，已有部署可以通过显式配置继续保持原来的服务发现方式。

### 配置管理体验与可靠性增强

Apollo 3.0 进一步完善了非 properties 格式的配置管理体验：

- JSON 配置支持格式化视图与原始内容视图；
- JSON、YAML 在保存链路进行严格语法校验；
- JSON、YAML、XML、TXT 等 Namespace 支持撤销未发布修改；
- OpenAPI 支持配置项批量新增、修改和删除。

这些能力既改善了 Portal 中的人工操作体验，也让自动化工具处理结构化配置时更加可靠。

## 升级注意事项

Apollo 3.0 是一次服务端技术基线和默认行为都发生变化的大版本升级，升级前请重点关注以下事项：

1. Apollo 服务端的 Java 运行基线已经从 Java 8 升级到 **Java 17**。
2. Config Service 和 Admin Service 默认切换为 `database-discovery`。已有 Eureka 部署如需保持原有行为，需要显式配置：

   ```bash
   SPRING_PROFILES_ACTIVE=github
   ```

3. 完整升级步骤、兼容性说明和全部变更，请参阅 [Apollo 3.0.0 Release Notes](https://github.com/apolloconfig/apollo/releases/tag/v3.0.0)。

## 欢迎共同建设

Agent First 对 Apollo 来说不是一个独立功能，也不是一个版本即可完成的终点，而是一条围绕接口、身份、授权、安全、工具和评测持续演进的产品方向。

Apollo 3.0 完成了这条道路上的第一轮基础建设。我们期待社区继续贡献真实的 Agent 使用场景、CLI 能力、评测任务、Agent Adapter，以及关于敏感配置保护、任务授权和审批治理的设计与实践。

让 Agent 能操作配置只是第一步。

让它在清晰的身份和安全边界内，可靠地完成正确的操作，才是 Apollo 迈向 Agent First 真正要解决的问题。
