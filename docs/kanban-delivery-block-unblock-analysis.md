# Kanban delivery blocked/unblock/completed 丢通知问题分析

## 结论

`own_v2026.6.19` 曾经存在 `delivery=true` 任务在 `blocked -> unblocked -> completed`
路径上丢 completed delivery 的问题。本分支已修复：`blocked` delivery 成功投递后只
推进 cursor，不再删除 agent delivery 订阅；后续任务 `completed` 后才退订。

根因不是事件没有写入，也不是 cursor rewind 失败，而是 `delivery_mode="agent"`
订阅在第一次成功投递 `blocked` 事件后被删除。任务后续 `kanban_unblock` 回到
`ready` 并最终 `completed` 时，`kanban_notify_subs` 已经没有对应订阅行，因此
不会再投递 completed 事件回原会话。

## 会话证据

会话文件：

`/Users/colsrch/Downloads/session-20260701_095823_c7b5b8b4.json`

关键事件链：

- `t_ad7b6f13` 由当前会话用 `kanban_create(..., delivery=true)` 创建。
- 该任务运行后进入 `blocked`，会话收到内部消息：
  `[Kanban task t_ad7b6f13 blocked - ... Reason: review-required: ...]`
- 当前会话处理 review 结果后调用 `kanban_unblock(task_id="t_ad7b6f13")`，
  返回 `{"ok": true, "task_id": "t_ad7b6f13", "status": "ready"}`。
- 会话最后一条 assistant 文案是“等待 `t_ad7b6f13` 最终完成通知”，但导出的
  session 中没有该任务的 completed delivery 消息。

这符合“blocked 能通知，unblock 后 completed 不通知”的现象。

## 修复前代码证据

普通 gateway notify 路径在 `gateway/kanban_watchers.py` 中已经有保护：
成功投递普通通知后，只在 task status 为 `done` 或 `archived` 时
`_kanban_unsub()`。这意味着普通 notify 不会因为 `blocked` 事件直接退订。

但是 `delivery_mode="agent"` 走的是另一条路径：

- `gateway/kanban_watchers.py::_handle_kanban_delivery()`
  成功注入事件后无条件执行：
  `self._kanban_advance(...)`，然后 `self._kanban_unsub(...)`。
- `tui_gateway/server.py::_poll_tui_kanban_deliveries_once()`
  成功 `_run_prompt_submit(...)` 后也无条件执行：
  `_tui_kanban_advance(...)`，然后 `_tui_kanban_unsub(...)`。

这两处都会在 `blocked` delivery 成功后删除订阅。

修复后，这两处成功路径都会先 advance cursor，再通过终态判断决定是否退订：

- 事件列表中包含 `completed`，退订；
- 或任务当前 status 是 `done` / `archived`，退订；
- 只有 `blocked` 这类非终态 delivery 时保留订阅。

不可恢复路径保持原行为：未知平台、source metadata 缺失、无法生成文本仍会
advance 后退订；adapter/session busy 或注入异常仍会 rewind，避免丢事件。

## 测试覆盖缺口

当前 `tests/gateway/test_kanban_notifier.py` 有参数化测试
`test_kanban_delivery_injects_synthetic_message_and_removes_sub`，覆盖
`completed` 和 `blocked` 两种 event_kind，并断言投递后
`kb.list_notify_subs(conn, tid) == []`。

当前 `tests/test_tui_gateway_server.py` 只覆盖 completed 后删除订阅、busy 时
rewind claim，没有覆盖：

1. blocked delivery 后订阅应保留；
2. unblock 后 completed 应继续投递；
3. completed 后才删除 agent delivery 订阅。

因此现有测试通过不能证明这个问题已修复；相反，现有测试把错误行为固定住了。

## 修复方案

已让 `delivery_mode="agent"` 与普通 notify 的生命周期一致：

- `blocked` delivery 成功后只 advance cursor，不删除订阅；
- `completed` delivery 成功后 advance cursor 并删除订阅；
- 无法生成文本、未知平台、source metadata 缺失这类不可恢复路径可以继续删除订阅；
- adapter/session busy 或 handle failure 仍应 rewind cursor。

已新增回归测试：

- gateway delivery：创建 agent delivery sub，先 `block_task` 并跑 notifier，
  断言收到 blocked 且订阅仍存在；随后 `unblock_task`、`complete_task`，
  再跑 notifier，断言收到 completed 且订阅被删除。
- TUI delivery：同样验证 blocked 后订阅保留、completed 后删除。

## 验证结果

已在 2026-07-01 跑过以下验证：

- `uv run --with pytest pytest tests/gateway/test_kanban_notifier.py::test_kanban_delivery_keeps_sub_after_blocked_until_completed tests/test_tui_gateway_server.py::test_tui_kanban_delivery_poller_keeps_sub_after_blocked_until_completed`
  - 结果：`2 passed`
- `uv run --with pytest pytest tests/gateway/test_kanban_notifier.py tests/test_tui_gateway_server.py`
  - 结果：`293 passed`
- `uv run --with pytest pytest tests/tools/test_kanban_tools.py`
  - 结果：`93 passed`
- `uv run --with ruff ruff check gateway/kanban_watchers.py tui_gateway/server.py tests/gateway/test_kanban_notifier.py tests/test_tui_gateway_server.py`
  - 结果：`All checks passed!`
