# Feishu kanban topic delivery routing regression analysis

## Conclusion

`own_v2026.6.19` missed the old Feishu topic delivery routing fix from
`feat/feishu-kanban-topic-delivery` (`2606b18b0 fix: route kanban delivery to
Feishu topics`). This has been fixed in the current branch.

The symptom in `/Users/colsrch/Downloads/session-20260701_153729_81aa7745.json`
is consistent with that missing fix:

- The Hermes session did receive the internal kanban delivery message:
  `[Kanban task t_cb109926 blocked - ...]`.
- The agent then generated a normal assistant reply saying the validation
  succeeded.
- That proves the kanban delivery was injected into the agent session.
- It does not prove Feishu displayed the message in the original topic.

For Feishu topic groups, backend "send succeeded" can still be invisible in
the expected topic if the send is not anchored to the triggering Feishu message
with `reply_in_thread=true`.

## Missing old fixes

The old branch changed two things:

1. `gateway/platforms/feishu.py`
   - Preserved the inbound Feishu `message_id` on `SessionSource`.
   - Old patch added `message_id=message_id` when building the source.

2. `gateway/run.py:_source_for_kanban_delivery()`
   - Preferred the live cached `SessionSource` over persisted
     `session_store.origin`.
   - This mattered because persisted origins can be stale and may not carry the
     latest topic reply anchor.

In current `own_v2026.6.19`, the relevant code moved into
`gateway/kanban_watchers.py::_source_for_kanban_delivery()`, but the old
priority rule was not ported. Current behavior still reads persisted
`session_store.origin` first and falls back to a reconstructed source.

Current `gateway/platforms/feishu.py` also builds the inbound source without
`message_id=message_id`, so the live cached source cannot carry the Feishu
reply anchor either.

## Why this breaks Feishu topic delivery

Kanban delivery creates a synthetic `MessageEvent` and injects it through the
platform adapter. In the gateway path, the synthetic event's `message_id` comes
from `source.message_id`.

Feishu topic sending needs both:

- `thread_id`, so Hermes knows this is a topic lane;
- `reply_to_message_id`, so Feishu's reply API can send with
  `reply_in_thread=true`.

Without `source.message_id`, the synthetic kanban event has no reply anchor.
The agent session still receives the message, but Feishu may route the outgoing
message outside the expected topic, or treat the send as successful without the
user seeing it in the topic thread being tested.

## Fix

The old branch's behavior has been ported to the current file layout:

1. Added `message_id=message_id` to Feishu inbound `build_source(...)` in
   `gateway/platforms/feishu.py`.
2. Updated `gateway/kanban_watchers.py::_source_for_kanban_delivery()` to prefer
   `self._get_cached_session_source(session_key)` before reading
   `session_store.origin`.
3. Added regression tests:
   - Feishu inbound topic message preserves `thread_id` and triggering
     `message_id`.
   - Kanban delivery prefers cached live source over stale persisted origin and
     passes the cached `message_id` into the synthetic event.

## Verification

Executed on 2026-07-01:

- `uv run --with pytest pytest tests/gateway/test_feishu.py::TestAdapterBehavior::test_process_inbound_topic_message_preserves_thread_and_trigger_message_id tests/gateway/test_kanban_notifier.py::test_kanban_delivery_prefers_cached_live_source_over_stale_session_origin`
  - Result before fix: `2 failed`
  - Result after fix: `2 passed`
- `uv run --extra dev --extra messaging pytest tests/gateway/test_feishu.py tests/gateway/test_kanban_notifier.py`
  - Result: `171 passed, 45 skipped`
- `uv run ruff check gateway/platforms/feishu.py gateway/kanban_watchers.py tests/gateway/test_feishu.py tests/gateway/test_kanban_notifier.py`
  - Result: `All checks passed!`
