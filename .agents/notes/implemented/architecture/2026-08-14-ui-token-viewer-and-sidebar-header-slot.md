# Agent Note: Token consumption surfaces and the sidebar header slot

Status: implemented

English | [中文](2026-08-14-ui-token-viewer-and-sidebar-header-slot.zh.md)

## Problem

Users want to see how many tokens a session (or the whole instance) has consumed, without leaving the web GUI. The host already computes exactly this: token-meter publishes `tokenUsage` / `contextPressure` / `contextBreakdown` as per-session projections, and the runtime publishes each session's projection values on the session-list rows. What is missing is a surface that reads them — and, for the sidebar, a place to mount one: the sidebar shell (`dsh-client-ui-sidebar`) declares only `sidebar.workspaces` (single), `sidebar.settings`, and `sidebar.footer.action`, so there is no slot above the workspaces region where a global card could live.

## Decision

Add a new client plugin package, `@deepseek-ai/dsh-client-ui-token-viewer`, with two projection-mode surfaces plus one host route (no store, no event listener; the only wire is the balance fetch):

- `TokenDock` registers into the existing `conversation.input.dock` list (order 20) and renders the current session's billed input, output, cache hit rate, and approximate context occupancy, with the full billing breakdown in a hover tooltip.
- `SidebarTokenPanel` registers into a new `sidebar.workspaces.header` hole declared by the sidebar shell and rendered above the workspaces region, aggregating `tokenUsage` across every session row's `projectionValues` (billed input, output, cache hit rate, reporting-session count). It hides in the collapsed rail. It also shows the DeepSeek account balance and expands to a per-conversation list: each session's billed input/output, highest total first, clicking a row opens that session through the injected `sessions` service.
- The node half registers `GET /api/billing/balance`: it resolves the `DEEPSEEK_API_KEY` credential reference through the credentials service (the same ref the LLM adapter uses) and proxies DeepSeek's `/user/balance`, returning only balance figures — the API key never leaves the server. The balance row fetches this route; failure surfaces an error-retry control, and an unconfigured key answers 503 `no-api-key` so the card hides rather than error-spamming.

The ui-sidebar change is additive and minimal: one child slot declaration (`kind: 'single', scope: 'root'`), one `renderSlot('sidebar.workspaces.header', { wide })` call above the workspaces region, and the matching `SlotMap` / owner-props / `PropsRenderSlots` contract entries. The hole is a platform seam, not a token-viewer special case: any global card that belongs above the browsing region can use it.

## Alternatives considered

**Fork the sidebar shell inside the token-viewer package and disable the stock row.** Works locally but duplicates the whole shell bundle, forces a profile-level `disabled` override, and would be rejected upstream: the duplicate `sidebar` declaration throws, and a forked bundle drifts from the shipped shell.

**Register into `sidebar.footer.action` (or `sidebar.settings`).** Both are the sidebar foot — the requested placement is above the workspaces region, and the foot carries interactive controls the card would crowd.

**Shadow `sidebar.workspaces` at a lower priority.** A single slot renders one entry; shadowing would replace the entire workspace browser, not add a card.

**Read `ctx.tokenMeter.measure()` directly on the client.** The meter service is host-plane; the browser reaches projections through the session projection store, which is exactly what these surfaces consume.

**Query the provider balance from the browser.** The API key is a server-side credential; a client-side call would expose it. The host route resolves the key through the credentials service and returns only balance figures.

## Consequences

Users see live per-session and whole-instance token consumption without leaving the GUI, from the same host-computed figures the compaction and occupancy displays use, plus the account balance from the provider's billing endpoint. The web profile gains one `dsh.client` row and dependency; `pnpm-lock.yaml` needs a regeneration after merge. The package is surface-only: it adds no prompt content, tools, messages, or provider requests, so there are no model or KV-cache effects (the balance route is a server-side fetch, not a model call). The balance is DeepSeek-specific — the route calls DeepSeek's `/user/balance` and shows only the first `balance_infos` entry. The sidebar card depends on the header hole; a composition replacing ui-sidebar without it silently loses the card while the dock strip keeps working (documented under Known Limitations).

The detail panel's model statistics fold a host `modelUsage` session projection over `assistant/message` events: each step's provider usage is attributed to its message model (`message.source.model`), and the browser half aggregates the per-session projection values into per-model rows (sessions, input/output/cache, estimated cost). It is a pure event fold with no model or KV-cache effects; until any session reports usage, the panel shows a single deployment-default-model row from the `agent-default-model` settings namespace (client `settingsScope` read, `deepseek-v4-flash` fallback).
