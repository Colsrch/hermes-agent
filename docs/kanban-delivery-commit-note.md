# Kanban Delivery Commit Note

## Commit

`feat(kanban): add delivery callbacks for gateway sessions`

## What It Changes

This commit adds an optional `delivery` parameter to the `kanban_create` agent
tool. When a gateway-hosted agent creates a Kanban task with `delivery=true`,
Hermes stores the originating platform session metadata with the task.

When that task later emits a `completed` or `blocked` event, the gateway Kanban
notifier injects a synthetic internal message back into the original session
through the platform adapter's normal `handle_message` flow.

## Why It Matters

This lets an agent fan out work to Kanban workers and then resume the original
platform conversation when the child task finishes or blocks. The final reply is
still produced by the normal gateway agent pipeline, so platform routing,
threading, session history, and adapter-specific reply behavior remain
consistent with regular user messages.

## Safety Notes

Delivery subscriptions are separate from human-facing Kanban notifications.
They are one-shot, cursor-protected, and removed after successful injection so
existing notification behavior stays isolated.
