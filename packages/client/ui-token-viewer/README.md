# @deepseek-ai/dsh-client-ui-token-viewer

English | [中文](README.zh.md)

Token consumption surface plugin, browser half: two read-only surfaces over the host-computed token-meter session projections (`tokenUsage`, `contextPressure`, `contextBreakdown`), so the plugin owns no domain store, refresh chain, or event listener.

- **`TokenDock`** registers at `conversation.input.dock` (order 20, after Goal). It shows what the current session has consumed — billed input (uncached + cache read + cache write), output, cache hit rate, and approximate context occupancy (`projectedTokens / contextWindow`) with a mini progress bar. The hover tooltip carries the full billing breakdown. It renders nothing until a provider reports usage.
- **`SidebarTokenPanel`** registers at `sidebar.workspaces.header`, a hole declared by ui-sidebar's shell above the workspaces region. It aggregates `tokenUsage` across every session row's `projectionValues` — billed input, output, cache hit rate, and the number of sessions that reported usage — so it is a whole-instance total rather than a per-session read. It renders nothing until a session reports usage, and nothing in the collapsed rail (`wide === false`).

The `/client` exports are the plugin body (`apply`/`inject`) and the composed props types.

## Model Experience

None. The surfaces are pure presentation over projection values already computed by the host; they add no prompt content, tools, messages, or provider requests.

#### KV Cache effect

None. The plugin neither assembles nor sends provider requests.

## Known Limitations and Deferred Work

- **Heuristic approximations** — cache hit rate and context occupancy inherit the token-meter's fixed 4-chars-per-token density estimate for any content the provider did not bill; CJK text and JSON schemas are systematically underpriced. Occupancy is a user-facing reference figure, not a billing or gating input (see the token-meter README).
- **The sidebar card depends on ui-sidebar's header hole** — it renders only when the shell declares `sidebar.workspaces.header`; a composition that replaces ui-sidebar without that hole silently loses the card while the dock strip keeps working.
