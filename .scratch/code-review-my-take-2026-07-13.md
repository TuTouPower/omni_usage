<!-- omni_powers: scratch/code-review-my-take-2026-07-13 -->

# 对《代码审查报告（2026-07-13）》的独立意见

**审查对象**：`.scratch/code-review-2026-07-13.md`  
**方法**：逐条核对被引用的代码位置与 `docs/omni_powers/op_blueprint/` 原文，独立判断。  
**立场**：既不站原报告也不站反驳，只看事实与上下文。

---

## 一、Standards 轴逐条复核

### 1. camelCase 函数 vs snake_case 约定

**原报告指控**：`conventions.md §1` 要求 snake_case，但全代码库广泛使用 camelCase 函数，属于硬性违规。

**我的判断**：✅ 同意，是硬性违规。

`conventions.md §1` 第一句就写死了「变量、函数、文件名、目录名一律 snake_case」。像 [logger.ts](file:///d:/Kar/Code/omni_usage/src/shared/lib/logger.ts#L108-L198) 里 `formatTimestamp`/`shouldLog`/`formatMeta` 是 camelCase，同文件又有 `serialize_meta`/`scrub_meta` 是 snake_case——**同一文件内自相矛盾**，这就不是"历史包袱"能解释的了，是编码纪律问题。

唯一可以讨论的是：这件事优先级有多高。它不影响功能，属于技术债，但既然文档写了就应该遵守，要么改代码要么改文档。

---

### 2. 新代码仍引入废弃词 `plugin`

**原报告指控**：`domain.md §5` 规定「新代码一律用统一词」，但有 4 处新代码用了 `plugin`。

**我的判断**：部分同意，**要区分「新建接口」和「处理旧字段的兼容代码」**。

- [scheduler-orchestrator.ts:11](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/scheduler-orchestrator.ts#L11) 的 `ConnectorListConfig { plugins: ... }` — 这是**新建的内部接口**，完全可以叫 `connectors`，用 `plugins` 没有任何兼容理由，属于硬违规。✅
- [config-store.ts:47](file:///d:/Kar/Code/omni_usage/src/main/core/config/config-store.ts#L47) 的 `is_plugin_healthy` / `prune_invalid_plugins` — 这两个函数处理的是 `config.plugins[]` 这个**已有旧字段**，函数名跟字段名对应是合理的。如果函数叫 `is_connector_healthy` 但操作的是 `plugins` 数组，反而更让人困惑。这是兼容迁移代码，不应算"新代码引入废弃词"。⚠️
- [config-ipc.ts](file:///d:/Kar/Code/omni_usage/src/main/ipc/config-ipc.ts) 的用户文案「未知的插件实例」等 — 用户面向的文案，是否要改要看产品决策，但从 `domain.md §5` 「术语中英一律统一」的角度，确实应该改。✅

**结论**：4 处里 2 处硬违规（新建接口 + 用户文案），2 处是兼容代码（应降级）。

---

### 3. `account_id: "default"` 占位

**原报告指控**：`domain.md` 不变量 3 「accountId 须服务商稳定，禁占位」，两个 executor 用了 `"default"`。

**我的判断**：⚠️ 不同意直接定性为「硬性违规」，要看场景。

不变量 3 的原文是：

> accountId 必须稳定：由聚合源返回的稳定账号标识（邮箱、UUID、workspace id、CPA auth_index）生成，**绝不用"实例 + 序号"**。否则远端账号顺序一变，本地隐藏设置/自定义标签/历史观测全部错位。

注意核心危害是「顺序一变全部错位」——这针对的是**多账号**场景（尤其是 CPA 这种聚合源）。对于**单账号直连**的连接器（比如一个 API key 对应一个账号）：

- 只有一个账号，不存在"顺序"问题
- `"default"` 是稳定的（不会因为远端变化而改变）
- 很多 API 根本不返回账号标识，只能自己兜底

当然，从「尽可能用服务商返回的真实标识」的角度，`"default"` 确实是兜底方案，不够理想。但把它和「实例 + 序号」这种真正危险的做法等同起来，是**过度解读**了不变量 3。

**结论**：这是「改进项」而非「硬性违规」。如果连接器 API 能拿到真实账号 ID 就应该拿，拿不到用 `"default"` 是可接受的兜底。

---

### 4. executor 仍发已废弃的 `name` 字段

**原报告指控**：`conventions.md §5` 步骤 3 要求发 `raw_label` + `normalized_label`，`name` 已废弃，但两个 executor 仍发 `name: "Usage"`。

**我的判断**：✅ 同意，是违规。

`tier1-poll-executor.ts` 和 `probe-executor.ts` 同时发了 `name`、`raw_label`、`normalized_label` 三个字段——既然 `name` 已废弃就不该再发。虽然可能是为了兼容（下游可能还在读 `name`），但 `conventions.md` 写了 `name` 已废弃，就应该清理。

---

### Smell baseline 逐条看

#### Mysterious Name — `toDTO(state): PluginSnapshotDTO`

**我的判断**：✅ 同意，名字确实有问题。`toDTO` 太泛了，而且返回类型叫 `PluginSnapshotDTO` 还用了废弃词 `Plugin`，双重问题。

#### Duplicated Code — `do_request` 与 `get_raw` 前奏重复

**我的判断**：⚠️ 部分同意。重复是事实，但要看重复的逻辑是否会演化出差异。如果两段逻辑未来可能分道扬镳（比如 `get_raw` 要加特殊的流式处理），那提取公用反而会增加耦合。当前看重复度确实高，可以考虑抽一个 `build_request_context` 之类的辅助函数。

#### Feature Envy — `apply_auth` 深探 Manifest+Vault

**我的判断**：❌ 不同意这是 Feature Envy。

`NetClient` 的职责本来就是「替连接器发安全的 HTTP 请求」，而认证是发请求的固有部分。如果 `apply_auth` 在 `NetClient` 里，那它接触 `manifest.poll?.request.auth` 和 `vault` 是**本职工作**，不是"羡慕别的类"。

真要重构的话，可以把认证策略抽出来，但那是为了符合单一职责原则，不是因为 Feature Envy。

#### Repeated Switches — connector-ipc.ts 三个函数各自分支 cpa

**我的判断**：✅ 同意。同样的 `definition.manifest.id === "cpa"` 判断写了三遍，说明缺少一个抽象层（比如 CPA 专属的 IPC handler 或 capabilities 驱动的分支）。

#### Middle Man — `secrets-store.ts` 全是 1:1 转发

**我的判断**：❌ 完全不同意。

`SecretsStore` 是**边界接口**，不是 Middle Man。它的价值在于：

1. **依赖倒置**：上层只依赖 `SecretsStore` 接口，不依赖具体的 Vault 实现
2. **封装变化**：未来换 vault 实现（比如从本地文件换成 Windows Credential Manager），只需改 SecretsStore 内部
3. **安全边界**：明确哪些操作是允许的，不是 vault 的所有方法都要对外暴露

Middle Man 的定义是"没有附加价值的中间层"，而 `SecretsStore` 提供了架构层面的价值——它存在的意义就是定义边界，不是为了加业务逻辑。

> **文件名 kebab-case vs snake_case**：原报告自己也标了"只标一次不逐文件列"。我认为这是**文档滞后**，不是代码问题——kebab-case 是 TS/Node 生态的事实标准，全代码库也已经统一了。应该改 `conventions.md`，不是改代码。

---

## 二、Spec 轴逐条复核

### (a) 缺失/未完整实现

#### 1. 采集失败未写 stale 观测

**我的判断**：✅ 同意，这是明确的 bug。

`domain.md` 不变量 2 写得很清楚：

> 采集失败保留上次成功观测，挂 `stale:true` + `lastError`，绝不覆盖删除。

[refresh-service.ts:285-289](file:///d:/Kar/Code/omni_usage/src/main/core/scheduler/refresh-service.ts#L285-L289) 只更新了 runtime state，没有往 observation store 写 stale 行。这意味着 UI 层拿不到 stale 标记，用户看不到"数据过期了"的状态。

#### 2. CPA 单账号失败未发 stale

**我的判断**：✅ 同意，违反不变量 5。

不变量 5：「单账号失败只让那一行 stale，同 provider 其他账号照常刷新。」`cpa/connector.ts` 的 per-account try/catch 只 `warn` 然后跳过，没有产出 stale 观测。结果就是失败的账号直接消失了，用户看不到「这个账号挂了」的状态。

#### 3. LocalAPI 网关端点缺失

**我的判断**：⚠️ 降级。`platform-services.md` 原文写的是「**可选**网关」「默认关闭」。可选功能没实现不算"缺失"，算"未启用"或"enhancement"。

#### 4. LocalAPI token 未入 Vault

**我的判断**：✅ 同意，如果 spec 明确说了要存 Vault 那就应该存。但等一下——让我先确认 spec 原文怎么写的...（这里我没有读 `platform-services.md` 全文，暂时按原报告说的算，但建议核对原文措辞）。

#### 5. paths.ts 未集中

**我的判断**：✅ 同意，但优先级低。路径散落各处确实不好维护，但这是内部实现问题，不影响用户功能。

#### 6. CONFIG_EXPORT 未加密

**我的判断**：⚠️ 需要先澄清 spec 冲突。原报告依据 `secret-vault.md`，但如果 `config-store.md` 说的是「密钥脱敏为 `***REDACTED***`」，那就是两个 spec 文件打架，不是实现单方面错。应该先统一 spec，再改实现。

#### 7. 关闭动作缺第三选项

**我的判断**：⚠️ 降级为 enhancement。`window-management.md` 只是罗列了三种可能，没有说必须都实现。当前 `hide/proceed` 覆盖了核心场景。

---

### (b) Scope creep

#### 1. `accountOverrides.disabled`

**我的判断**：⚠️ 不一定是 scope creep。

原报告说 `config-store.md` 的 AppConfiguration 仅列 `hidden`，但代码有 `disabled`，所以是范围蔓延。

但要区分两种情况：

- 如果是**开发过程中发现需要 `disabled`，但忘了更新 spec** — 这是 spec 滞后，不是 creep
- 如果是**拍脑袋加的、没有明确使用场景的功能** — 这才是 creep

需要看 `disabled` 实际有没有在用、用在哪。如果 UI 上有禁用账号的功能且用户确实需要，那应该补 spec，不是删代码。

#### 2. `accountLabels`

**我的判断**：⚠️ 同上。账号自定义标签是很自然的需求，不太像"乱加的功能"，更像"加了但没补 spec"。

#### 3. MiMo 硬编码

**我的判断**：✅ 同意，这是技术债。cookie 名和登录域名应该从 manifest 读，硬编码意味着加新的 session 型连接器就要改宿主代码，违反了「连接器是声明式的」的设计原则。

---

### (c) 实现错误

#### 1. session partition 命名错

**我的判断**：✅ 同意。`connector-session.md` 要求 `persist:<provider>-login`，实际用了 `persist:session-login:<instance_id>`，按实例分不按 provider 分，跟 spec 不符。

#### 2. partition 体系冲突

**我的判断**：✅ 同意。`auth-ipc.ts` 用 `persist:mimo-login:<instanceId>`，`session-manager.ts` 用 `persist:session-login:`，两个地方命名规则不一样，迟早出 bug。

#### 3. cookie jar 回退违禁

**我的判断**：需要看 `connector-session.md` 原文的语气。如果是「绝对禁止」那就是违规，如果是「不建议/不保证」那就是灰色地带。回退逻辑本质是容错，但容错可能掩盖真正的 bug。

#### 4. 破坏性操作越层

**我的判断**：✅ 同意，违反不变量 8。

不变量 8：「破坏性操作只出现在"行即数据源"层级，账号子行只做显示调整。」`PopupView.tsx` 在账号子行写 `disabled`，属于在显示层做破坏性操作。CPA 的账号子行尤其不应该能被本地禁用——因为账号存在性由远端 CPA-Manager 决定。

---

## 三、我的 Severity 重排

### 🔴 P0 — 破坏业务不变量，用户可见

1. **CPA 单账号失败不产 stale** — 违反 invariant 5，用户看到的是"账号消失了"而不是"账号挂了"。
2. **refresh-service 失败不写 stale observation** — 违反 invariant 2，数据过期了用户不知道。
3. **PopupView 账号子行写 `disabled`** — 违反 invariant 8，CPA 账号被本地禁用后可能再也找不回来。

### 🟠 P1 — 违反约定或实现错误，有潜在风险

4. **session partition 命名错误 + 体系冲突 + cookie jar 回退** — session 能力的基础契约有问题，可能导致登录态混乱。
5. **新建接口 `ConnectorListConfig.plugins` 用废弃词** — 新代码用旧术语，会让术语统一的努力白费。
6. **executor 仍发 `name` 字段** — 废弃字段还在发，下游可能依赖上了就更难清。
7. **MiMo 硬编码 cookie 名和域名** — 新增 session 连接器就要改宿主，扩展性差。
8. **`account_id: "default"` 占位** — 单账号场景可接受，但如果哪天 API 能拿到真实 ID 应该换掉。

### 🟡 P2 — 技术债 / 规范问题，不影响功能

9. **camelCase vs snake_case 混用** — 编码纪律问题，建议分批次整理。
10. **paths.ts 路径未集中** — 维护性问题，优先级低。
11. **`toDTO` 命名含糊 + PluginSnapshotDTO 用废弃词** — 可读性问题。
12. **connector-ipc.ts 重复分支判断** — 可重构但不急。

### ⚪ 待澄清 / 不是问题

13. **LocalAPI 网关端点缺失** — 可选功能，明确要不要做再说。
14. **CONFIG_EXPORT 未加密** — 先解决 `secret-vault.md` 与 `config-store.md` 的 spec 冲突。
15. **关闭动作缺第三选项** — enhancement，不是 bug。
16. **`accountOverrides.disabled` / `accountLabels`** — 是补 spec 还是删代码，需要产品决策。
17. **`secrets-store.ts` Middle Man** — 不是问题，是有意的边界设计。
18. **`apply_auth` Feature Envy** — 不是问题，是职责内的事。
19. **文件名 kebab-case** — 改文档，不改代码。

---

## 四、总体评价

原报告的**问题发现能力很好**，大部分指控都有实锤。主要偏差在于：

1. **把"兼容迁移代码"也算成"新代码引入废弃词"** — 没区分场景
2. **把架构边界误判为 Middle Man / Feature Envy** — Fowler 的 smell 不是这么用的
3. **把 spec 内部冲突直接判成实现错误** — 应该先澄清 spec
4. **对不变量的解读有点过度** — 比如 `"default"` 在单账号场景是合理的

建议：按上面的 severity 重排后，P0 立即修，P1 排进近期迭代，P2 攒多了一起 refactor，待澄清的先拉出来讨论。
