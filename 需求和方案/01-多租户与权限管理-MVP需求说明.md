# 多租户与权限管理 MVP 需求说明

- 状态：待评审
- 版本：v0.1
- 适用项目：LightRAG
- 编写日期：2026-10-10

## 1. 背景

当前 LightRAG 已具备 `workspace` 级别的存储命名空间、部分共享状态隔离，以及 JWT/API Key 等部署级认证能力。但当前 workspace 主要是实例或存储配置，不是经过用户授权的资源；系统尚未形成用户、租户、workspace 成员关系和持久化权限模型。

因此，当前系统不能安全地向多个相互不信任的组织提供完整的空间隔离服务。目标不是简单增加一个请求头，而是建立从身份认证到数据访问的完整授权链路。

## 2. 术语

- **User**：系统用户，拥有可验证的身份。
- **Tenant/Organization**：组织级租户，隔离组织成员和资源。
- **Workspace/Space**：租户内部的数据空间，文档、图谱、向量、处理状态和查询上下文均归属于 workspace。
- **Membership**：用户与租户或 workspace 的成员关系。
- **Role**：成员角色。
- **Permission**：具体操作权限。
- **Principal**：认证后得到的调用主体，可为用户或受限 API credential。
- **WorkspaceContext**：服务端完成授权后形成的不可变工作空间上下文。

## 3. MVP 目标

1. 支持持久化用户身份和登录状态。
2. 支持创建、查看、归档和删除 workspace。
3. 支持用户加入多个 workspace，并为每个 workspace 分配角色。
4. 支持 workspace 级文档、查询、图谱和 pipeline 隔离。
5. 所有业务路由在访问数据前执行统一的身份、成员关系和权限校验。
6. 支持 owner、admin、editor、viewer 四种基础角色。
7. 支持关键操作审计。
8. 支持旧的单 workspace 部署平滑迁移。

## 4. MVP 非目标

第一期暂不实现：

- 文档级 ACL。
- 字段级或记录级权限。
- 跨 workspace 搜索和跨租户查询。
- 自定义 ABAC 策略编辑器。
- 多区域灾备。
- 任意存储后端的跨 Pod 多写者自动协调。
- 复杂的组织计费、配额和商业套餐系统。

## 5. 角色定义

| 角色 | 说明 |
|---|---|
| owner | workspace 所有者，可管理空间、成员、权限并执行高风险操作 |
| admin | 空间管理员，可管理成员和空间配置，但默认不能转移 owner 或删除租户 |
| editor | 可上传、处理、编辑业务数据，但不能管理成员和角色 |
| viewer | 只读访问，可查询和查看允许的文档/图谱 |

## 6. 核心业务规则

### 6.1 身份和授权

1. 未认证请求不得访问受保护业务接口。
2. 已认证不等于有权访问所有 workspace。
3. 权限来自服务端持久化 membership；客户端不能通过修改 JWT 的 role、header、URL、query 或 body 绕过授权。
4. API Key 必须绑定租户、workspace 或明确 scope；数据库只保存 hash。
5. 用户被禁用后，其已有 token 必须在规定时间内失效。

### 6.2 Workspace

1. workspace 使用不可变的内部 ID 和独立的 canonical storage namespace。
2. display name 和 slug 不能直接作为文件路径或存储主键。
3. 同一租户内 workspace slug 唯一。
4. 删除或归档 workspace 时，不得影响其他 workspace。
5. workspace 的所有文档、图谱、向量、缓存、doc-status、pipeline 状态和审计记录必须可追溯到该 workspace。

### 6.3 请求上下文

每个受保护请求必须按以下顺序处理：

```text
认证 token/API credential
  -> 得到 Principal
  -> 解析请求目标 workspace
  -> 查询并校验 membership
  -> 校验 permission
  -> 构造 WorkspaceContext
  -> 获取对应 RAG runtime
  -> 执行业务操作
```

客户端 workspace header 只允许作为兼容入口或目标提示，不得直接作为授权依据。

## 7. MVP 业务流程

### 7.1 创建 workspace

```text
用户登录
  -> 提交 workspace 名称
  -> 服务端校验创建权限和名称
  -> 创建 workspace 记录
  -> 创建 owner membership
  -> 生成 storage_namespace
  -> 初始化或登记 RAG runtime
  -> 写入审计事件
```

### 7.2 上传文档

```text
携带 token 请求上传
  -> 解析 Principal
  -> 解析 workspace_id
  -> 校验 document:insert
  -> 获取 workspace 对应的 RAG runtime
  -> 写入该 workspace 的存储
  -> 写入审计事件
```

### 7.3 跨空间访问

用户属于 workspace A，但请求 workspace B 时：

- 服务端不得读取 B 的文档、图谱、向量或状态。
- 根据资源存在性策略返回 403 或统一 404。
- 不得泄露 B 的文档数量、名称、处理状态和错误细节。
- 失败操作应写入审计记录。

## 8. 数据隔离验收标准

1. 用户 A 属于 workspace A，用户 B 属于 workspace B。
2. A 上传的数据不会出现在 B 的文档列表、查询结果、图谱和向量检索中。
3. A 修改 workspace 标识后，不能读取或修改 B。
4. 同名文档可以分别存在于 A 和 B，且不会覆盖。
5. 删除 A 不会删除 B 的数据。
6. 重启服务后，A/B 的数据仍保持正确归属。
7. workspace 初始化、迁移或失败恢复不会把数据写入其他 workspace。
8. workspace 相关的锁、pipeline 状态、缓存和审计记录不会跨空间复用。

## 9. 权限验收标准

1. viewer 可以查询，但不能上传、删除、编辑图谱或执行 pipeline。
2. editor 可以上传、处理和编辑业务数据，但不能管理成员。
3. admin 可以管理成员和空间配置，但不能执行 owner 专属操作。
4. owner 可以转移所有权或删除 workspace，具体高风险操作必须二次确认。
5. 禁用用户的旧 token 不能继续执行受保护操作。
6. 权限变更在规定的 token/session 生效窗口内生效。
7. 所有敏感成功和失败操作都包含 actor、workspace、action、resource 和结果。

## 10. 兼容与迁移要求

旧系统映射建议：

```text
旧 AUTH_ACCOUNTS -> 初始 users
旧 WORKSPACE/--workspace -> legacy tenant + default workspace
旧 rag_storage -> default workspace 的 storage_namespace
旧共享 API key -> legacy credential 或要求重新创建
```

迁移必须支持：

- dry-run
- 幂等执行
- 按 workspace 执行
- 状态记录和失败重试
- 备份与回滚
- 失败时 fail closed

旧单 workspace 模式可通过 feature flag 保留；新启用的多租户模式默认拒绝未授权访问。

## 11. MVP 交付边界

第一阶段建议只支持：

- PostgreSQL 控制面。
- JSON 文件后端和 PostgreSQL RAG 后端的 workspace 隔离验证。
- 四种基础角色。
- workspace 级文档、query、graph、pipeline 权限。
- 审计日志。
- legacy 单 workspace 迁移。

其他后端和高级权限模型按后续阶段逐个接入。
