# Issue #16328 Investigation: Subagent lifecycle leak after compaction

Upstream issue: https://github.com/openai/codex/issues/16328

## Summary

This note investigates a Codex CLI multi-agent runtime issue where, after session compaction, Codex may keep reporting `Agent spawn failed` even though earlier subagents have already completed.

The issue is valuable because it is not just a UI error. It points to a runtime lifecycle problem: completed child agents may remain counted as live execution capacity after compaction or resume.

## User-visible symptom

The reported behavior is:

1. A Codex CLI session spawns enough subagents to reach the configured subagent/thread limit.
2. Those subagents finish their work.
3. The session is manually compacted or auto-compacted.
4. In the same terminal session, Codex later attempts to spawn more subagents.
5. Codex repeatedly reports `Agent spawn failed`.
6. Opening a new terminal tab and resuming from the same repository may avoid the issue.

This suggests the current compacted session may still treat completed subagents as active capacity consumers.

## Why this matters for agent runtime design

For long-horizon coding agents, subagents are not just chat messages. They are runtime resources with lifecycle state.

The key design distinction is:

- **Restorable history**: old subagent threads and their outputs should remain inspectable or recoverable.
- **Live execution slots**: only currently active or reusable runtime workers should count against the spawn/thread cap.

A completed subagent can remain restorable without continuing to occupy a live spawn slot.

## Relevant code areas

The likely relevant files are:

- `codex-rs/core/src/agent/control.rs`
- `codex-rs/core/src/agent/registry.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_spec.rs`
- thread state / rollout / compaction related code paths

## Initial code reading

### Spawn slot reservation

`AgentControl::spawn_agent_internal` reserves a slot before creating a child thread:

```rust
let mut reservation = self.state.reserve_spawn_slot(config.agent_max_threads)?;
```

After the thread is created, the reservation is committed:

```rust
reservation.commit(agent_metadata.clone());
```

This registers the spawned thread in `AgentRegistry`.

### Registry counting model

`AgentRegistry::reserve_spawn_slot` increments `total_count` when a max thread limit is configured.

`AgentRegistry::release_spawned_thread` removes an agent from `agent_tree` and decrements `total_count` if the removed agent was counted.

### Release paths

`release_spawned_thread` is called in paths such as:

- `handle_thread_request_result` when `InternalAgentDied` occurs
- `shutdown_live_agent` after shutdown and removal from `ThreadManagerState`

However, a normal completed subagent does not necessarily imply that its slot is released. Completion notification and slot release appear to be separate concepts.

This distinction is probably intentional in some cases because a completed child may still be reusable or restorable. But it creates a risk: if completed agents stay in the active registry after compaction, the runtime may hit the thread cap even though no useful live work is happening.

## Root-cause hypothesis

The bug is likely not simply "spawn failed". It is more likely a lifecycle accounting mismatch:

```text
completed child thread
    -> preserved as open/restorable history
    -> still present in live agent registry or open thread-spawn edge
    -> still counted against max_concurrent_threads_per_session / agent_max_threads
    -> later spawn fails even though previous work is done
```

In short:

> The runtime may be conflating historical/restorable subagent records with live execution slot accounting.

## Correct lifecycle model

A more precise lifecycle model should separate these states:

```text
PendingInit     counts against live slot
Running         counts against live slot
Waiting         counts against live slot
Completed       does not need to count against live slot unless explicitly kept alive for follow-up
Released        does not count against live slot, but metadata/history may remain
Restorable      persisted history exists, but no active runtime slot is reserved
Closed          explicitly closed and should not be auto-restored as open
Failed          terminal failure, should not leak slot
```

## Proposed fix direction

The safest design is not to delete completed child history aggressively.

Instead:

1. Keep completed subagent history and metadata restorable.
2. Release live spawn capacity when a child reaches a final state and no longer needs to remain active.
3. Ensure compaction/resume does not restore completed children as live active children unless explicitly requested.
4. Keep `close_agent` as the explicit user-facing cleanup path, but do not require the model/user to manually close already-finished children just to free capacity.

A possible implementation direction:

- Add a helper such as `release_finished_spawned_thread_if_final(thread_id)`.
- Call it from the completion watcher after final status notification is delivered.
- Or introduce a distinct persisted edge status such as `Completed` / `Released`, instead of treating all non-closed child edges as `Open`.
- Make resume/compaction restore only genuinely open live children, not all completed historical children.

## Regression test idea

A useful test should reproduce the lifecycle leak without relying on a real model:

1. Configure a low subagent/thread limit, e.g. 2 or 3.
2. Spawn child agents until the cap is reached.
3. Mark or drive those child agents to final `Completed` status.
4. Trigger compaction or simulate resume from compacted history.
5. Attempt another spawn from the same root/session.
6. Expected: spawn succeeds because completed children no longer occupy live capacity.
7. Also assert that completed child history remains available for inspection or resume when requested.

## Related issue: #22779 and existing fix branch

After starting from #16328, I found a newer related issue: #22779, `Completed subagents continue to count against thread limit`.

That issue has an existing fix branch by `pengyou200902`:

- https://github.com/openai/codex/issues/22779
- https://github.com/pengyou200902/codex/tree/fix/subagent-thread-limit

The branch addresses the same core lifecycle/accounting problem more directly.

## Review of the existing fix direction

The existing branch separates two concepts:

1. retained agent metadata
2. counted live execution quota

The key design idea is:

```text
agent metadata exists
    does not necessarily mean
agent still consumes a live spawn slot
```

This matches the core runtime principle from this investigation:

```text
Restorable history should be separate from live execution-slot accounting.
```

The branch also handles reuse: if a completed agent receives new work later, it should reacquire a quota slot before execution.

## Review questions / possible risks

Important cases to check:

1. Releasing quota should be idempotent. Repeated final-status events must not decrement the counter twice.
2. Reusing a completed agent should reacquire quota before starting new work.
3. If reacquiring quota fails, task state should not be partially updated.
4. Resume and compaction should not restore completed agents as counted live agents.
5. `list_agents` semantics should stay clear: listed/restorable is not the same as counted/running.
6. `close_agent` should still remove metadata and release any remaining counted slot.

## Tradeoff

The main tradeoff is between:

- freeing resources automatically; and
- preserving the ability to inspect or resume older subagents.

The clean solution is not to choose one. The runtime should preserve historical/restorable thread records while separately tracking live execution capacity.

## Interview framing

This issue can be explained as:

> I investigated a Codex multi-agent runtime bug where completed subagents may remain counted as active after compaction, causing later `spawn_agent` calls to fail. The main insight is that an agent runtime needs to separate historical/restorable child-thread state from live execution-slot accounting. A completed subagent should remain inspectable, but it should not leak active capacity.

After finding #22779, I shifted from duplicating the same patch to reviewing the existing fix direction. That is closer to real engineering work: identify the root cause, compare related reports, credit existing work, and evaluate the tradeoffs.

This maps to broader agent-system design topics:

- long-horizon agent state management
- subagent lifecycle and orchestration
- compaction and state restoration
- runtime reliability beyond prompt engineering
- regression testing for agent workflow correctness

## Next steps

- Locate the exact test harness around `AgentControl`, `AgentRegistry`, and `ThreadManagerState`.
- Review the existing branch's tests around completed-agent slot release and slot reacquisition.
- Decide whether any smaller follow-up contribution is still useful, such as documentation, a focused regression test, or a clearer lifecycle comment.
- Avoid duplicating the same fix branch unless maintainers ask for an alternative implementation.
