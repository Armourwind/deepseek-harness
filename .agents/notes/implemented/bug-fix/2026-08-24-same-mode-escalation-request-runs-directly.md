# Agent Note: Same-mode escalation requests run directly

Status: implemented

English | [中文](2026-08-24-same-mode-escalation-request-runs-directly.zh.md)

## Problem

The escalation argument pairing could not tell a same-mode request from a real escalation. `validateEscalationArgs` validated `sandbox_permissions` and `justification` without knowing the call's standing sandbox mode, and `approveEscalation` rejected any request whose target was not strictly wider. A caller asking for the mode already in force — which widens nothing — therefore fell into a fail-closed dead end: no justification produced `invalid justification` or `requires a justification`, and a justification produced `not strictly wider than this call's current mode`. In a session whose current mode was already `danger-full-access` with approval prompts disabled, every call that carried a permission field at all hit one of these rejections, so the model could not recover by retrying without the field (its tool-call generation kept supplying one), and `write`/`edit` — the same-family tools that happened to be called without the field — worked, making the failure look like a tool-schema defect. The schema and the parameter-transmission chain were correct; the validator simply had no concept of "already effective".

## Decision

A request for the mode already in force is NOT an escalation, and it never reaches the approval channel. The shared validator gained an optional `effectiveMode` argument: `validateEscalationArgs(sandboxPermissions, justification, effectiveMode)` returns immediately when `effectiveMode` is defined and `sandboxPermissions === effectiveMode`, skipping the pairing and non-empty-justification checks. Each enforcing family passes its standing mode in: `tool-bash` and `tool-pwsh` resolve the standing policy before validating and short-circuit same-mode requests past `approveEscalation` in `execute`; `tool-fs`'s `FsSandboxController.resolvePolicy` resolves the standing policy first, validates with `standingPolicy.mode`, and stamps the standing policy directly when the request is same-mode.

The other three branches keep their fail-closed behavior. A request for a strictly wider mode still requires a non-empty justification and goes through `approveEscalation` (with approval unavailable or rejected, it still fails closed). A narrower request is still rejected as `not strictly wider`. A malformed empty-string `sandbox_permissions` is still rejected by the JSON-Schema enumeration check (`ToolArgsError`) before execute — that is a caller generation defect, not something the tool layer admits.

## Consequences

- The dead end is gone: a same-mode request executes directly regardless of whether a justification was supplied, in `bash`, `pwsh`, `write`, and `edit` alike.
- No permission is ever widened by the new path: same-mode means the standing mode, and the policy stamped onto the call is the standing policy.
- Three pre-existing assertions that pinned the old same-mode rejection (one each in the `tool-bash` and `tool-pwsh` generic-producer suites, plus the `tool-fs` pairing test, which now uses a strictly wider target to exercise the pairing rule) were updated to the new behavior, and four regression assertions were added: the shared validator admits a same-mode request with no or blank justification, and both the bash and the fs families execute a same-mode request without invoking `approval/request`.
- Callers whose generation inserts an empty-string `sandbox_permissions` still see `must be one of ["workspace-write","danger-full-access"]`; that is the JSON-Schema layer rejecting a malformed value, and the model-side fix is to omit the field or pass a real mode, which the same-mode path now accepts.

## Alternatives considered

Keep the fix in a host adapter (the model-facing `functions.bash` layer) instead of the shared validator: the same dead end affects `tool-fs` and `tool-pwsh`, so a one-place shared fix covers every enforcing family, and there is no adapter code in this checkout to change.

Teach the model, through tool descriptions, never to carry permission fields: descriptions already say to omit them, yet generation kept supplying them; a prompt-level rule cannot make a same-mode retry succeed and leaves the validator unable to distinguish same-mode from widening.

Relax the JSON-Schema check to tolerate an empty string: that weakens the enum contract for a malformed value that the tool layer has no business admitting, and it does nothing for same-mode requests carrying a valid mode.
