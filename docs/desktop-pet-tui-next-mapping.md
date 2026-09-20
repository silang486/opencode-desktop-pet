# Desktop Pet `tui-next` replacement mapping

Status: pre-implementation design. The existing Python/Textual TUI is frozen. This document describes the boundary for a new client; it does not authorize changes to `F:\VPet-GitHub\tools\desktop-pet-studio` yet.

Research baseline:

- Fork: `https://github.com/silang486/opencode-desktop-pet`
- Upstream: `https://github.com/anomalyco/opencode`
- Fork branch: `codex/pet-tui-next`
- Baseline commit: `ebb7b76eca82342642c78645109e865614533827`
- TUI package: `packages/tui`, package name `@opencode-ai/tui`, version `1.18.31`, private workspace package, MIT licensed.
- Current implementation uses Solid/OpenTUI and the workspace client/schema packages. It is not an npm package that can be installed independently.

## Decision

Reuse the OpenCode TUI application shell and its client-side state projection. Keep the Desktop Pet Agent, LangChain `create_agent`, Skills, typed production tools, H3/GPU workers, matting, petpack and development worker in Python. A Python HTTP/SSE adapter becomes the only boundary visible to the TUI.

The TUI must never import `pet_agent.py`, LangChain message classes, tool objects, graph nodes, provider clients or GPU modules. The Python side must never call Solid/OpenTUI components. The only shared contract is the versioned client API, schemas and event stream.

## Current TUI to OpenCode mapping

| Current surface | Current implementation | OpenCode basis | `tui-next` decision |
|---|---|---|---|
| Application shell | `tools/desktop-pet-studio/tui.py::DesktopPetApp` | `packages/tui/src/app.tsx`, route/context providers | Reuse the OpenCode shell and provider composition; remove the Textual shell from the new path. |
| Session sidebar | `ListView`, `ClientController.session_rows()` | `routes/session/sidebar.tsx`, `dialog-session-list.tsx`, session data store | Reuse sidebar/navigation. Add PetProject and active-run indicators through adapter data. |
| Conversation | `RichLog`, `history_count`, manual event index | `routes/session/index.tsx`, Message/Part projection in `context/sync.tsx` and `context/data.tsx` | Reuse timeline, scrolling and parts. Convert Python events into Message and Part records. |
| Composer | Textual `TextArea`, custom bindings | `component/prompt/index.tsx`, prompt history/stash/autocomplete/local attachment | Reuse composer, multiline editing, history, paste and focus behavior. Map ask/steer/queue to explicit submission metadata. |
| Stop | `ClientController.command('/stop')` plus thread liveness | `sdk.client.session.abort()` and session status events | Use an idempotent abort API. Unlock the composer from server status/event facts, not Python thread liveness. |
| Rename/fork/edit | custom right-click and `ClientController.fork()` | session rename/fork dialogs and timeline actions | Reuse dialogs and actions; old messages and artifacts remain immutable. |
| Tool progress | `tool_progress` string rendered into one `Static` | tool Parts and tool display utilities | Use one Part per tool call with pending/running/completed/failed/cancelled state and structured progress. |
| Todo | state field plus custom detail panel | `Todo`, `todo.updated`, `sidebar/todo`, `todo-item` | Use OpenCode Todo projection. The backend emits real plan changes only. |
| Permission | no stable request object | permission context and `routes/session/permission.tsx` | Reuse permission dialog. Add production-specific permission names and risk metadata. |
| Artifact preview | `open_artifact_in_tui`, custom ASCII/image preview | OpenCode links/attachments and dialog primitives | Add a domain Artifact Part and a preview adapter; keep media paths behind API responses. |
| Background jobs | `RunControl`/telemetry mixed into controller | session status, background job events and notifications | Expose `GenerationJob` as a first-class domain record and event stream. |
| Theme and keyboard | local Textual CSS and bindings | `theme`, `keymap`, `ui`, `dialog` packages | Use the OpenCode visual language and keymap. Do not create a second theme system. |

## Components to reuse directly from the fork

The preferred unit of reuse is the package/workspace source, not copied snippets:

- `packages/tui/src/app.tsx` and provider composition;
- `packages/tui/src/routes/session/*` for session layout, sidebar, footer, permission and question flows;
- `packages/tui/src/component/prompt/*` for the composer, history, stash, autocomplete and attachments;
- `packages/tui/src/component/dialog-*` and `packages/tui/src/ui/*` for dialogs, command palette, toasts and selection controls;
- `packages/tui/src/context/sdk.tsx`, `context/event.ts`, `context/data.tsx`, `context/sync.tsx` for API/event hydration and projection;
- `packages/tui/src/util/scroll.ts`, `selection.ts`, `transcript.ts`, `tool-display.ts` and `presentation.ts`;
- `packages/tui/src/theme/*`, `keymap.tsx`, clipboard and terminal adapters;
- `packages/client` generated client contracts and `packages/schema` event/message schemas where a compatible subset is practical.

The package is private and depends on workspace packages such as `@opencode-ai/client`, `@opencode-ai/schema`, `@opencode-ai/core`, `@opentui/*` and `solid-js`. The first implementation task is therefore a fork-local build and adapter, not an npm import.

## Components that need adaptation

- Replace the OpenCode server URL/directory assumptions with a Desktop Pet backend URL and project/session scope.
- Implement the client contract in Python, with HTTP JSON requests and a reconnecting SSE event stream carrying cursor/event IDs.
- Project OpenCode `Message` and `Part` records from Python durable events. Each record must carry `sessionID`; assistant/tool parts also carry `runID`, `taskID` and request version where applicable.
- Map Desktop Pet tool effects to OpenCode-like Tool Parts. The TUI does not decide whether an operation is paid, destructive, development-only or safe; it renders permission requests emitted by the backend.
- Map `PetProject`, `Character`, `Action`, `GenerationJob`, `Artifact`, `GPUJob` and `ReviewRequest` into domain data returned by the adapter and rendered with the same dialog, list, Part and status components.
- Preserve OpenCode reconnect/hydration behavior: subscribe before requesting the initial snapshot, replay events after a cursor, then reconcile authoritative session/messages/parts/todos/jobs.
- Keep Windows clipboard, image paste and terminal sizing behind client/runtime adapters. Do not reintroduce Python widget-specific clipboard or focus code into the TUI.

## Desktop Pet-only components

Only these concepts should be added to the domain layer:

- `PetProject`: selected product, profile status, current mode and project revision;
- `Character`: profile/reference/master lineage and approval state;
- `Action`: PIC2/VIDEO2/H3 action intent and dependency state;
- `GenerationJob`: provider request, cost/allowance state, queue state and submission-unknown state;
- `Artifact`: immutable prompt/image/video/matte/petpack record, preview metadata and lineage;
- `GPUJob`: remote queue ID, provider state, polling/receipt state and cancellation capability;
- `ReviewRequest`: approval, rejection, conflict or missing evidence requiring the user.

These are data and events, not a second UI language. They should appear as OpenCode-style Parts, dialogs, notifications, sidebar rows and status blocks.

## Backend API and event contract

The adapter should implement a versioned `/api/desktop-pet/v1` surface. The names below are the minimum contract; exact wire schemas must be generated and checked from one source of truth.

### Requests

- `GET /sessions` — list/search/resume sessions with status and unread event count.
- `POST /sessions` — create a session.
- `GET /sessions/{sessionID}` — authoritative session metadata and current status.
- `PATCH /sessions/{sessionID}` — rename/archive/project binding with optimistic revision.
- `GET /sessions/{sessionID}/messages?after=...` — durable Message and Part history.
- `POST /sessions/{sessionID}/prompt` — admit an input with `delivery: ask | steer | queue`, client message ID and attachments.
- `POST /sessions/{sessionID}/abort` — idempotent local abort request; returns `aborting` plus affected run/task IDs.
- `POST /sessions/{sessionID}/permission/{requestID}` — reply once/always/reject with an explicit scope.
- `GET /sessions/{sessionID}/todo` — actual plan state.
- `GET /sessions/{sessionID}/jobs` — GenerationJob/GPUJob state and receipts.
- `GET /artifacts/{artifactID}` — metadata and safe preview/download URL.
- `GET /events?after={cursor}` — reconnectable SSE stream.

All mutating calls return a durable identifier and current revision. They must be safe to retry with the same client request ID.

### Event envelope

```json
{
  "id": "evt-...",
  "cursor": "...",
  "type": "session.next.tool.progress",
  "sessionID": "session-...",
  "runID": "run-...",
  "taskID": "task-...",
  "messageID": "msg-...",
  "partID": "part-...",
  "at": "2026-09-20T10:00:00Z",
  "properties": {}
}
```

Required event families:

- session created/updated/status changed;
- user prompt admitted/promoted and assistant step started/ended/failed;
- assistant text started/delta/ended;
- tool input started/delta/ended, called, progress, success, failed, cancelled;
- todo updated;
- permission asked/replied;
- background job queued/running/completed/failed/cancelled/submission-unknown;
- artifact created/approved/rejected;
- review requested/resolved;
- non-active session notification.

`aborted` is an execution fact emitted after the backend acknowledges the abort boundary. `aborting` is not treated as completed. A new prompt may be admitted according to its delivery mode without waiting for a remote provider cancellation, but the old run may not write results into the new run.

## Mock backend required before real Python wiring

The mock must be a protocol server, not a fake TUI callback. It must emit deterministic fixtures for:

1. create/list/switch/resume and rename;
2. user message plus assistant text deltas;
3. queued/running/completed/failed/cancelled Tool Parts;
4. two concurrent tool calls with interleaved progress;
5. aborting then aborted, including a delayed provider result that is ignored;
6. Todo updates;
7. permission request and reply;
8. background GenerationJob updates;
9. Artifact creation and preview metadata;
10. an event for a non-active session that increments attention without changing the active transcript.

The first TUI acceptance tests should run entirely against this mock and assert rendered state, focus, scroll position, composer availability, event replay and no cross-session contamination.

## Migration plan

1. Freeze the existing Python/Textual TUI at its current revision and add a `legacy_tui` launch path. No more feature patches go into it.
2. Keep the OpenCode fork pinned to a recorded upstream commit and make `packages/tui` build/test in isolation from the fork workspace.
3. Add a fork-local `tui-next` entrypoint and protocol fixture/mock server. Do not connect it to the Python Agent yet.
4. Implement the Desktop Pet adapter contract and event projection against the mock. Make the TUI pass the parity scenarios before real backend work.
5. Add a Python HTTP/SSE server adapter over existing SessionStore, RunControl, durable events, tools and workers. This adapter owns translation; existing Agent/tools remain behind it.
6. Connect real read-only session/history/status flows, then prompt/stream/abort, then tools/permissions/jobs/artifacts. Each phase gets replay and reconnect tests.
7. Add `--tui-next` as an opt-in launch path and run real Windows terminal acceptance beside legacy_tui. Keep legacy available as fallback until feature parity is proven.
8. Cut over the default only after mock, protocol, real backend and Windows E2E acceptance pass. Remove legacy code only in a separate cleanup change after the cutover evidence is archived.

## Parity gate

Parity requires: session create/list/switch/resume; rename/fork/edit; multiline composer and paste; streaming; scroll/focus/resize; tool Part rendering; concurrent jobs; stop/abort/retry semantics; Todo; permissions; artifact preview/open; background jobs; inactive-session notifications; reconnect/replay; Chinese Windows input; and no imports from Python implementation modules inside the TUI package.

The current Textual TUI is not used as the architectural base for any of these pieces. It remains only as a frozen fallback and behavioral reference during migration.
