# Kanban timeout delivery analysis

## Conclusion

`timed_out` is not a task status in the Kanban task table. It is a
`task_runs.outcome` / `task_events.kind`.

The task itself is blocked by the circuit breaker after repeated failures. The
dashboard may surface the latest run outcome (`timed_out`), which makes it look
like the card status is `timed_out`, but the task status model is:

- `triage`
- `todo`
- `scheduled`
- `ready`
- `running`
- `blocked`
- `review`
- `done`
- `archived`

## Why delivery does not fire

Gateway ordinary notifications and agent delivery use different event filters.

Ordinary notify mode watches:

- `completed`
- `blocked`
- `gave_up`
- `crashed`
- `timed_out`

Before this fix, agent delivery mode (`delivery=true`) watched only:

- `completed`
- `blocked`

So a task that emitted `timed_out`, then tripped the circuit breaker and emitted
`gave_up`, was not injected back into the originating agent session via
`delivery=true`.

## Why the example timed out immediately

The error:

`elapsed 60s > limit 0s`

means the task row had `max_runtime_seconds = 0`. In current code, `None` means
"no runtime cap"; `0` is stored as an integer cap and therefore acts as an
immediate timeout once the dispatcher checks the task.

## Fix

This change fixes the delivery side:

- Gateway agent delivery now claims and injects:
  - `completed`
  - `blocked`
  - `gave_up`
  - `crashed`
  - `timed_out`
  - `spawn_failed`
  - `protocol_violation`
  - `rate_limited`
- TUI agent delivery now uses the same event set.
- Both paths now render explicit synthetic messages for exception events so the
  originating agent session can react to abnormal task outcomes.

The `max_runtime_seconds = 0` normalization/rejection is still a separate
follow-up. This fix ensures the originating agent learns about the timeout and
circuit-breaker failure instead of silently missing it.

For an already-blocked task, fix the underlying runtime cap first, then reclaim
or unblock the task. If the task still has `max_runtime_seconds = 0`, retrying
will reproduce the same timeout.

## Verification

Executed on 2026-07-01:

- `uv run --with pytest pytest tests/gateway/test_kanban_notifier.py::test_kanban_delivery_injects_task_exception_events tests/test_tui_gateway_server.py::test_tui_kanban_delivery_poller_delivers_task_exception_events`
  - Before fix: `7 failed`
  - After fix: `7 passed`
- `uv run --with pytest pytest tests/gateway/test_kanban_notifier.py tests/test_tui_gateway_server.py`
  - Result: `301 passed`
- `uv run --with pytest pytest tests/tools/test_kanban_tools.py`
  - Result: `93 passed`
- `uv run --with ruff ruff check gateway/kanban_watchers.py tui_gateway/server.py tools/kanban_tools.py tests/gateway/test_kanban_notifier.py tests/test_tui_gateway_server.py`
  - Result: `All checks passed!`
