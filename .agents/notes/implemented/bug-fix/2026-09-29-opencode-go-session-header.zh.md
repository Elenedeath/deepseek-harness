# Agent Note: OpenCode Go 请求以 seam 会话 id 作为路由头

Status: implemented

[English](2026-09-29-opencode-go-session-header.md) | 中文

## Problem

OpenCode Go 按请求头 `x-opencode-session` 对会话进行路由与提示词缓存。pi-ai 0.87.1 已为目录中的 `opencode-go` provider 原生标记该头，但以其他 provider 名指向 Go 端点的手声明路由由 adapter 的协议表构建，pi-ai 不为其包裹该头，因此这类请求不带该头发出。于是这类路由上的每个会话都共享 provider 的默认路由与提示词缓存，一个会话的上下文可能被投递给另一个会话。

## Decision

`dsh-llm-pi-ai` 在 adapter 中派生该头：对于携带 seam `sessionId` 的 `opencode-go` 请求，stream 调用发送以该 id 为值的 `x-opencode-session`。部署配置的 `x-opencode-session` profile 头优先生效，按大小写不敏感匹配；未携带会话 id 的请求不发送该头。派生逻辑位于既有的 profile 头合并处，Harness 拥有的归因头名在那里本就赢得冲突。

## Alternatives considered

**打补丁让 pi-ai 目录为 opencode-go 声明会话亲和性。** 不采纳：目录由上游拥有且被固定，compat 标志会让 pi-ai 自行编造头值，而不是使用主循环、压缩与会话标题调用共同共享的 Harness 会话身份。

**为所有 provider 发送该头。** 不采纳：路由语义属于 OpenCode Go，把 Harness 会话 id 盖到其他部署的请求上会在没有意义的地方泄露内部身份。

**以会话（conversation）id 作为头的键。** 不采纳：seam 会话 id 才是贯穿一次会话的主循环、压缩与会话标题调用的稳定身份，恰好是 OpenCode Go 路由所依据的分组。

## Consequences

一个 Harness 会话现在发送一个稳定的 `x-opencode-session` 值，OpenCode Go 据此将其路由并缓存为一个会话。已配置该头的部署保持其值不变，未携带会话 id 的请求不发送该头，该路径保持此前的行为。在目录路由上，adapter 的派生先于 pi-ai 的原生标记，而后者会跳过已存在的头，因此两者共存且不重复。

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` 用 mock 服务器固定四种情况：携带会话 id 的 `opencode-go` 请求发送派生的头；携带同一 id 的 `deepseek` 请求不发送；部署配置的 `X-Opencode-Session` profile 头按大小写不敏感地优先；未携带会话 id 的 `opencode-go` 请求不发送。
