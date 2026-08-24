# Agent Note: Same-mode escalation requests run directly

Status: implemented

[English](2026-08-24-same-mode-escalation-request-runs-directly.md) | 中文

## Problem

升级参数配对校验无法区分"同模式请求"与"真实升级"。`validateEscalationArgs` 在不知道调用当前沙箱模式的情况下校验 `sandbox_permissions` 与 `justification`，而 `approveEscalation` 拒绝一切目标不是严格更宽的请求。调用方请求恰好是当前已生效模式（不提升任何权限）时，因此落入 fail-closed 死结：不带 justification 报 `invalid justification` 或 `requires a justification`，带 justification 报 `not strictly wider than this call's current mode`。在"当前模式已是 danger-full-access 且审批提示被禁用"的会话里，任何携带权限字段的调用都会命中其中一种拒绝，模型即使重试不携带该字段也无法恢复（其工具调用生成仍会补上该字段），而恰好未被携带权限字段调用的同族工具 `write`/`edit` 正常工作，让故障看起来像工具 schema 缺陷。schema 与参数透传链本身正确；问题只是校验器缺少"已生效模式"这一概念。

## Decision

请求已生效模式的调用**不是升级**，也绝不进入审批通道。共享校验器新增可选参数 `effectiveMode`：当 `effectiveMode` 已定义且 `sandbox_permissions === effectiveMode` 时，`validateEscalationArgs(sandboxPermissions, justification, effectiveMode)` 立即返回，跳过配对与非空 justification 校验。各 enforcing 族传入其当前模式：`tool-bash` 与 `tool-pwsh` 先解析 standing policy 再校验，并在 `execute` 中将同模式请求短路绕过 `approveEscalation`；`tool-fs` 的 `FsSandboxController.resolvePolicy` 先解析 standing policy，用 `standingPolicy.mode` 校验，同模式请求直接盖章返回 standing policy。

其余三分支保持 fail-closed 行为不变。请求严格更宽的模式仍需非空 justification 并走 `approveEscalation`（审批不可用或被拒时仍然关闭）。更窄请求仍被以 `not strictly wider` 拒绝。畸形空串 `sandbox_permissions` 仍在 execute 之前被 JSON-Schema 枚举校验（`ToolArgsError`）拒绝——那是调用方生成缺陷，工具层不放行。

## Consequences

- 死结消除：`bash`、`pwsh`、`write`、`edit` 中，同模式请求无论是否携带 justification 都直接执行。
- 新路径绝不提升权限：同模式即当前模式，盖章到调用上的 policy 就是 standing policy。
- 三条固定旧"同模式被拒"行为的既有断言（`tool-bash` 与 `tool-pwsh` 的 generic-producer 套件各一条，`tool-fs` 配对校验测试现改用严格更宽目标来覆盖配对规则）已更新为新行为；另新增四条回归断言：共享校验器接受无/空白 justification 的同模式请求，bash 与 fs 两族执行同模式请求时不触发 `approval/request`。
- 生成时插入空串 `sandbox_permissions` 的调用方仍会看到 `must be one of ["workspace-write","danger-full-access"]`；那是 JSON-Schema 层拒绝畸形值，模型侧修复方式是省略该字段或传入真实模式值——后者现在会被同模式路径接受。

## Alternatives considered

只在宿主适配层（模型可见的 `functions.bash` 层）修复而不动共享校验器：同样的死结影响 `tool-fs` 与 `tool-pwsh`，一处共享修复即可覆盖全部 enforcing 族，且本 checkout 中不存在可改的适配层代码。

通过工具描述教导模型绝不携带权限字段：描述本已说明省略，但生成仍会补上；提示层规则无法让同模式重试成功，也令校验器无法区分同模式与升级。

放宽 JSON-Schema 以容忍空串：这会弱化枚举契约，放行工具层本不应接受的畸形值，且对携带合法模式值的同模式请求毫无帮助。
