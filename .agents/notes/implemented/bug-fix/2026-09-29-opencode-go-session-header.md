# Agent Note: OpenCode Go requests carry the seam session id as their routing header

Status: implemented

English | [中文](2026-09-29-opencode-go-session-header.zh.md)

## Problem

OpenCode Go routes and prompt-caches a conversation by the `x-opencode-session` request header. pi-ai 0.87.1 stamps that header natively for the catalog `opencode-go` provider, but a hand-declared route pointed at the Go endpoint under another provider name is built from the adapter's protocol table, which pi-ai does not wrap, so such requests went out without the header. Every conversation on such a route therefore shared the provider's default route and prompt cache, and context from one conversation could be served to another.

## Decision

`dsh-llm-pi-ai` derives the header in the adapter: for an `opencode-go` request that carries the seam's `sessionId`, the stream call sends `x-opencode-session` set to that id. A deployment-configured `x-opencode-session` profile header wins, matched case-insensitively, and a request without a session id sends no header. The derivation sits next to the existing profile-header merge, where Harness-owned attribution names already win collisions.

## Alternatives considered

**Patch the pi-ai catalog to declare session affinity for opencode-go.** Rejected: the catalog is upstream-owned and pinned, and a compat flag would make pi-ai invent the header value itself instead of using the Harness session identity that main, compaction, and session-title calls all share.

**Send the header for every provider.** Rejected: the routing semantics are OpenCode Go's, and stamping Harness session ids onto other deployments' requests would leak internal identity where it has no meaning.

**Key the header on the conversation id.** Rejected: the seam session id is the stable identity across a conversation's main, compaction, and session-title calls, which is exactly the grouping OpenCode Go routes by.

## Consequences

One Harness conversation now sends one stable `x-opencode-session` value, so OpenCode Go routes and prompt-caches it as one conversation. A deployment that already configures the header keeps its value unchanged, and a request without a session id sends none, preserving the previous behavior for that path. On catalog routes the adapter's derivation precedes pi-ai's native stamping, which skips a header that is already present, so the two coexist without duplication.

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` pins the four cases against a mock server: an `opencode-go` request with a session id sends the derived header; a `deepseek` request with the same id sends none; a deployment-configured `X-Opencode-Session` profile header wins case-insensitively; and an `opencode-go` request without a session id sends none.
