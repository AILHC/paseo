# Agent lifecycle

How an agent is created, runs, becomes a subagent, gets archived, and disappears from the UI. The model spans the daemon (lifecycle, archive) and the client (tabs, the subagents track).

## States

```
initializing → idle → running → idle (or error → closed)
                 ↑        │
                 └────────┘  (agent completes a turn, awaits next prompt)
```

Each agent in `AgentManager` carries a `lastStatus` of `initializing`, `idle`, `running`, `error`, or `closed`. State transitions persist to disk and stream to subscribed clients via WebSocket.

## Relationships

Agents can launch other agents via the agent-scoped `create_agent` MCP tool. Agent-scoped creation is always asynchronous. By default, the daemon stamps the created agent with a label `paseo.parent-agent-id` pointing back at the agent that created it. The client surfaces that as `agent.parentAgentId`.

Agent-scoped `create_agent` accepts `detached: true` for agents that should stand on their own. The daemon still uses the creating agent for cwd/config inheritance, but does not write `paseo.parent-agent-id`.

- **Subagents** — created with `detached: false` or omitted. They exist as part of the creating agent's work, appear in that agent's subagent track, and are archived with it.
- **Detached agents** — created with `detached: true`. They take over as sibling/root agents (e.g. handoffs, fire-and-forget delegations), do not appear in the creating agent's subagent track, and are not archived with it.

`notifyOnFinish` defaults to `true` for agent-scoped creation because most subagents are delegated work the creating agent needs to hear back from. Set it to `false` only for truly fire-and-forget agents.

## Archive

Archive is a **soft delete**: the agent record stays on disk with `archivedAt` set, the runtime is closed, and the agent disappears from active lists. Archive is **global** — it lives on the server and propagates to every connected client.

`create_agent_request` can opt an agent into `autoArchive`. In that mode the daemon archives the agent after the first terminal turn event (`turn_completed`, `turn_failed`, or `turn_canceled`). If the same request created a Paseo worktree through its `worktree` field, auto-archive archives that worktree too, which removes the agent records inside the worktree.

Explicit session archive runs through `AgentManager.archiveSession` (`packages/server/src/server/agent/agent-manager.ts`). Other server-side archive flows still use `AgentManager.archiveAgent`, which delegates to the same archive boundary.

1. Snapshot the current session into the registry
2. Set `archivedAt` and normalize `lastStatus` away from `running`/`initializing`
3. Notify subscribers
4. Close the runtime (kills the process if still running)
5. **Cascade-archive children** — any agent whose `paseo.parent-agent-id` label matches the archived agent gets archived too, recursively

Cascade is what keeps subagent fleets from outliving their orchestrator.

### Server-side archive session boundary

The legacy `archive_agent_request` RPC is disabled for Web-originated archive requests.
Official Web clients may still send this request when closing tabs, but the server does not
treat it as an archive action. Use `/archive-session` to explicitly archive the current
session.

`close_items_request` ignores agent ids and only performs terminal close work. This keeps
official Web tab close behavior layout-only without requiring Web code changes.

Native unarchive runs through the shared server path before clearing Paseo's own
`archivedAt`. Codex uses the app-server `thread/unarchive` request. If native unarchive
fails with a real provider error, Paseo leaves the local record archived; idempotent
already-unarchived or not-archived results are accepted.

When the agent directory is refreshed, the server can ask providers for archived persisted
sessions and sync only non-live stored records into Paseo's archived state. Live agents are
not marked archived by that background sync.

## Tabs vs archive

These are two distinct concepts that used to be conflated:

| Concept                    | Scope      | Triggers                   |
| -------------------------- | ---------- | -------------------------- |
| **Tab** (workspace layout) | Per-client | User opens/closes a view   |
| **Archive** (lifecycle)    | Global     | Explicit lifecycle gesture |

Closing a Web tab is layout-only from the server's perspective. Web-originated legacy archive
requests fail and bulk close requests ignore agents. To archive any agent, send
`/archive-session` to that agent.

Closing a tab on a **subagent** (any agent with `parentAgentId`) is also **layout-only**.
The agent stays unarchived and stays in its parent's track. The user can re-open the tab
from the track at any time.

## Workspace activity

Agent lifecycle status stays literal: a parent agent is `idle` when its own turn is idle, even if a child is running.

Workspace status is an aggregate activity signal. Root agents contribute their normal state bucket to their own workspace. Running subagents contribute `running` to their root parent's workspace, not to the subagent's current `cwd` or worktree. Non-running subagent attention, permission, and error states stay in the parent's subagents track and do not escalate the workspace bucket.

## The subagents track

The collapsible track above the composer in an agent's pane (`packages/app/src/subagents/track.tsx`). Membership rule (`packages/app/src/subagents/select.ts`):

```
parentAgentId === thisAgent.id  AND  !archivedAt
```

Archived subagents disappear from the track, by design. To remove a subagent from the track without closing its tab, use the **archive button (X)** on the row — it opens a confirm dialog and archives the subagent on confirm. That same archive shows the subagent leave the track on every connected client.

## Why this shape

The decision was to **decouple "close tab" from "archive" at the server boundary**:

- **Closing a Web tab is layout-only** — server-originated archive state is not changed by Web close requests
- **`/archive-session` is explicit** — lifecycle archive is a deliberate command, not a side effect of layout
- **Subagent track membership remains archive-based** — completed children stay visible until explicitly archived
- **Cascade archive on parent** — keeps subagents from leaking when the parent is archived

This avoids changing official Web code while preventing tab close from archiving provider-native
sessions such as Codex threads.

## Limitations

### Subagent accumulation under long-lived parents

A parent that spawns many subagents will see the track grow. There's no automatic cleanup for completed subagents — the user prunes via the archive button on each row. A bulk gesture (e.g. "archive all idle children") could land later if this becomes a real problem.

### Cross-client tab dismissal

Closing a subagent's tab on one client doesn't affect other clients' layouts. This is the expected behavior of decoupled tabs and is consistent with how layouts have always worked. Archive remains the global gesture for cross-client cleanup.

## Storage

```
$PASEO_HOME/agents/{cwd-with-dashes}/{agent-id}.json
```

Each agent is a single JSON file. Fields relevant to this doc:

| Field                             | Type          | Meaning                                                                                   |
| --------------------------------- | ------------- | ----------------------------------------------------------------------------------------- |
| `id`                              | `string`      | Stable identifier                                                                         |
| `archivedAt`                      | `string?`     | Soft-delete timestamp (ISO 8601)                                                          |
| `labels["paseo.parent-agent-id"]` | `string?`     | Parent agent ID, set automatically by agent-scoped `create_agent` unless `detached: true` |
| `lastStatus`                      | `AgentStatus` | `initializing` / `idle` / `running` / `error` / `closed`                                  |

See [`docs/data-model.md`](./data-model.md) for the full agent record.
