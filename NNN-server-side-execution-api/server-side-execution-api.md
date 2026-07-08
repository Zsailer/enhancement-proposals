---
title: Server-Side Execution REST API
authors: <list-of-authors>
issue-number: <pre-proposal-issue-number>
pr-number: <proposal-pull-request-number>
date-started: "2026-07-08"
---

# Server-Side Execution REST API

## Summary

This JEP defines a standard REST API for **server-side execution** of notebook
cells: a client asks the server to run one or more cells by ID, and the server
drives the kernel on the client's behalf. Execution is *fire-and-forget* over
HTTP — the endpoint validates and enqueues the request, then returns
immediately. Kernel outputs are **not** returned in the HTTP response; they are
written back into the shared document and reach clients out-of-band (in the
reference implementations, over the real-time collaboration channel).

The immediate goal is to codify the single endpoint that several projects have
independently converged on — `POST /api/kernels/{kernel_id}/execute` — as a
first, votable version. Once accepted, the intent is to add this API to
**Jupyter Server core**, and to align existing implementations (notably
[jupyverse](https://github.com/jupyter-server/jupyverse)) behind it.

This JEP specifies **v1**. It is expected to evolve. The **canonical,
living definition of the API is the OpenAPI specification maintained in the
[`jupyter-server/jupyter_server`](https://github.com/jupyter-server/jupyter_server)
repository**. Where this document and that OpenAPI spec ever disagree after
this JEP is accepted, the OpenAPI spec is authoritative.

## Motivation

Traditionally, a Jupyter cell is executed by the *client*: the browser holds a
kernel WebSocket connection, sends an `execute_request` Jupyter message, and
routes the resulting `iopub` messages back into the notebook UI. This works, but
it ties execution to a live browser connection and to client-side message
routing. It becomes awkward for use cases such as:

- **Server-owned documents / real-time collaboration**, where the authoritative
  notebook state (including outputs) lives on the server and is synced to all
  clients. Here it is natural for the *server* to run the kernel and write
  outputs directly into the shared document.
- **Headless, agent-driven, or programmatic execution**, where there may be no
  browser at all, or where a non-notebook client wants to trigger execution of
  cells in a document that other clients are viewing.
- **Robustness across reconnects**, where execution should survive a client
  losing and re-establishing its connection.

Several independent projects have already converged on the *same* pattern to
solve this: a small REST endpoint that triggers execution by cell ID, with
outputs flowing back through the document-sync channel rather than the HTTP
response. This includes:

- [jupyverse](https://github.com/jupyter-server/jupyverse) (see
  [jupyverse#191](https://github.com/jupyter-server/jupyverse/pull/191)),
- [jupyter-server-nbmodel](https://github.com/datalayer/jupyter-server-nbmodel),
- the `pycrdt-websocket` + `jupyter_server_ydoc` stack from
  [jupyter-collaboration](https://github.com/jupyterlab/jupyter-collaboration)
  (whose `NotebookCellServerExecutor` speaks this contract), and
- [jupyter-server-documents](https://github.com/jupyter-ai-contrib/jupyter-server-documents)
  (see [jupyter-server-documents#248](https://github.com/jupyter-ai-contrib/jupyter-server-documents/pull/248),
  the reference implementation this JEP is derived from).

The problem is that this shared pattern is *convention*, not *specification*.
Each project re-derives the request shape, the field names, the response codes,
and the ordering guarantees. Small divergences (a renamed field, a different
hash, a different status code) are enough to break interoperability between a
frontend and a server that both believe they implement "server-side execution."

The expected outcome of this JEP is a single, agreed contract that:

1. can be implemented by any Jupyter server (Jupyter Server, jupyverse, …), and
2. can be relied upon by any client (JupyterLab, Notebook 7, custom frontends,
   automation), and
3. ultimately ships in **Jupyter Server core**, so it is available by default
   rather than reinvented per project.

## Background

### Client-side vs. server-side execution

In the classic model the client is the execution driver:

```
Browser ──execute_request──▶ Kernel
Browser ◀──iopub outputs──── Kernel
Browser ──writes outputs──▶ Notebook model (in the browser)
```

In the server-side model the *server* is the driver, and the document is the
delivery channel for outputs:

```
Client ──POST /execute (cell ids)──▶ Server
                                     Server ──execute_request──▶ Kernel
                                     Server ◀──iopub outputs──── Kernel
                                     Server ──writes outputs──▶ Shared document
All clients ◀── document sync ─────────────────────────────── Shared document
```

The HTTP request in the server-side model is only a *trigger*. It carries no
outputs back. This is the key architectural distinction and the reason the API
is fire-and-forget: the response says "I accepted this," not "here is the
result."

### The role of the shared document

Server-side execution is most natural when notebook state lives on the server
and is synchronized to clients — the model used by real-time collaboration
(RTC), where the document is a [Yjs](https://yjs.dev/) CRDT (a "YDoc") synced
over a WebSocket. In that world, the server writing an output into the YDoc cell
map is *sufficient* to deliver it to every connected client, with no bespoke
output-routing layer.

This JEP does **not** require RTC. It requires only that the server deliver
outputs by updating the shared document that clients are observing. RTC/YDoc is
named throughout as the *reference* mechanism because that is what the existing
implementations use, but the API is deliberately transport-agnostic about how
document updates reach clients (see
[Output delivery](#output-delivery-transport-agnostic)).

### Identifying cells and detecting drift

Because the request references cells *by ID* rather than by sending source
inline, and because other users may be editing the same document concurrently,
a client must be able to say "run the cell as it read when I pressed Run." The
API does this with a per-cell **source hash**: the client sends a hash of the
cell source it intends to execute, and the server refuses (with `409`) if the
document's current source for that cell no longer matches. This prevents the
classic race where a user runs a cell, a collaborator edits it a moment later,
and the kernel executes something the requester never saw.

## Guide-level explanation

Suppose a user opens a collaborative notebook and clicks **Run All**.

Instead of the browser sending three `execute_request` messages over a kernel
WebSocket, the frontend makes a single HTTP call:

```http
POST /api/kernels/8f3a.../execute
Content-Type: application/json

{
  "document_id": "json:notebook:5c1ffa...",
  "cells": [
    { "cell_id": "c1", "source_hash": "3531899427" },
    { "cell_id": "c2", "source_hash": "199471982"  },
    { "cell_id": "c3", "source_hash": "884012933"  }
  ],
  "client_id": "1837465",
  "request_id": "a1b2c3d4-..."
}
```

The server looks up the live document `json:notebook:5c1ffa...`, verifies that
the current source of `c1`, `c2`, and `c3` still hashes to the values the client
sent, and enqueues all three cells **atomically and in order**. It then responds:

```http
200 OK
Content-Type: application/json

null
```

That `null` means "accepted." The user's browser does not learn the results from
this response. Instead, as the kernel runs each cell, the server writes
`execution_state`, `execution_count`, and each output directly into the shared
document. Every client viewing that notebook — including the one that pressed
Run — sees the outputs appear through normal document synchronization.

**How to think about it:**

- **Frontend authors** think of `execute` as "tell the server to run these
  cells." They do not wire up an output listener for the HTTP call; outputs
  arrive through the document they are already rendering.
- **Server implementers** think of `execute` as "validate, enqueue, return."
  The hard part — pulling `iopub` messages off the kernel and writing them into
  the document — is theirs, but it is decoupled from the HTTP request lifecycle.
- **Ordering is explicit.** For a plain "Run All," listing the cells in one
  request is enough (the batch is enqueued atomically). For sequences of
  *separate* requests that must stay ordered, the client threads
  `request_id`/`previous_request_id` so the server preserves FIFO order even if
  the HTTP calls arrive out of order.

If a collaborator edits cell `c2` in the instant between the click and the
request, the server returns:

```http
409 Conflict
Content-Type: application/json

{ "error": "source_mismatch", "cell_id": "c2" }
```

and the frontend can re-read the cell and prompt the user rather than silently
running stale code.

## Reference-level explanation

This is the technical specification of **v1**. Field names, types, and status
codes are normative unless explicitly marked otherwise. The canonical machine-
readable form is the OpenAPI spec in `jupyter-server/jupyter_server`; this
section describes the same contract in prose.

### Endpoint

```
POST /api/kernels/{kernel_id}/execute
```

- `{kernel_id}` is the ID of an existing kernel, as returned by the standard
  `/api/kernels` API. The kernel must already exist; this endpoint does not
  start kernels.
- The endpoint requires authentication and is subject to Jupyter Server's
  authorization layer under a dedicated auth resource named **`executions`**
  (so deployments can grant or deny server-side execution independently of other
  kernel operations).
- Request and response bodies are JSON (`Content-Type: application/json`).

### Request body

| Field                 | Type                       | Required | Description |
|-----------------------|----------------------------|----------|-------------|
| `document_id`         | string                     | **yes**  | Identifier of the shared document ("room") that contains the cells. In the reference implementations this is the collaboration room name, e.g. `json:notebook:<file_id>`. The document must be live/loaded on the server. |
| `cells`               | array of cell objects      | **yes**  | One or more cells to execute, **in the given order**. Must be non-empty. |
| `cells[].cell_id`     | string                     | **yes**  | ID of the cell within the document. |
| `cells[].source_hash` | string                     | **yes**  | Hash of the cell source the client intends to run (see [Source hash](#source-hash)). |
| `client_id`           | string                     | no       | The requesting document client's ID (e.g. the CRDT/`doc.clientID`). Used for attribution and awareness. |
| `request_id`          | string (UUID)              | no       | Client-generated UUID identifying this execution request. Enables ordering and correlation. |
| `previous_request_id` | string (UUID)              | no       | UUID of the request that must be enqueued *before* this one. The server waits until that request's batch is fully enqueued before enqueuing this one, providing a FIFO guarantee across independent HTTP calls. |

Notes:

- **Batch atomicity.** All cells in `cells` are validated (hash-checked) and
  enqueued as a single atomic unit before the response is sent. No other
  request may interleave within a batch. This makes "Run All" and "Restart and
  Run All" correct regardless of network timing.
- **Ordering within a batch** is the array order of `cells`.
- **Ordering across batches** is unspecified *unless* the client uses
  `request_id`/`previous_request_id`. When those are provided, the server
  enqueues in the specified chain order.

### Source hash

`source_hash` is a string that lets the server detect that a cell's source has
drifted since the client decided to run it. In this v1 specification the hash is
**MurmurHash2 with seed `0`, encoded as a decimal string** — matching the
reference implementation and the hash the reference frontend computes over the
cell source.

MurmurHash2 is a fast, non-cryptographic hash. Its purpose here is *drift
detection*, not security: the server only needs to cheaply decide "is the source
the same as what the client saw?" A mismatch yields `409` (see below).

> **Evolution note.** The choice of hash is the most implementation-coupled part
> of this spec and the most likely to change (see
> [Unresolved questions](#unresolved-questions)). Implementers should treat the
> exact algorithm as pinned by the canonical OpenAPI spec, and clients and
> servers must agree on the same algorithm to interoperate.

### Responses

| Status | Body | Meaning |
|--------|------|---------|
| `200 OK` | `null` | **Accepted.** The batch was validated and enqueued. This is *not* a statement that execution has completed or succeeded — it is fire-and-forget. |
| `400 Bad Request` | error message | Malformed request: missing `document_id`, empty/invalid `cells`, a cell missing `source_hash`, or no live document for `document_id`. |
| `408 Request Timeout` | error message | The request specified `previous_request_id`, and that predecessor did not become enqueued within the server's wait window. |
| `409 Conflict` | `{ "error": "source_mismatch", "cell_id": "<id>" }` | A cell's current source in the document does not match its `source_hash`. `cell_id` identifies the first offending cell. No cells in the batch are enqueued. |

The `200` body is literally the JSON value `null`. Clients should treat any
`2xx` with a `null`/empty body as "accepted"; the specific `null` payload is
retained for compatibility with existing implementations (jupyverse returns
`null`).

### Semantics

**Fire-and-forget.** The HTTP response carries no execution results. A `200`
means the server has taken ownership of running the listed cells. Progress and
results are observed through the document, not the response.

**Validation before side effects.** The server validates the entire batch
(document exists, is a notebook document, every cell resolvable, every hash
matches) *before* enqueuing anything. On any `4xx`, nothing is enqueued.

**Execution ordering.** Within a batch, cells run in array order. Across
batches, ordering is only guaranteed when the client chains requests with
`request_id`/`previous_request_id`; this lets a client fire multiple `execute`
calls without awaiting each response yet still guarantee the kernel sees them in
intent order (important because HTTP requests can otherwise race).

**Clearing outputs.** When a cell is accepted for execution, its previous
outputs are cleared in the document and `execution_state`/`execution_count` are
updated as execution proceeds. (The reference implementation clears outputs at
enqueue time.)

### Output delivery (transport-agnostic)

This is the one deliberately **non-normative-mechanism** part of the spec.

**Normative requirement:** a conforming server MUST deliver kernel outputs by
updating the **shared document** identified by `document_id` — writing outputs,
`execution_count`, and execution state into the corresponding cells — such that
clients observing that document receive them. Outputs MUST NOT be expected in
the HTTP response.

**Reference mechanism (non-normative):** the existing implementations deliver
document updates over the **real-time collaboration channel** — the document is
a Yjs CRDT (YDoc) and the server writes iopub outputs into the YDoc cell map,
which propagates to clients over the collaboration WebSocket. This JEP names RTC
as the reference because it is what is deployed today, but does **not** require a
server to use Yjs, a WebSocket, or any particular sync technology. A server that
delivers outputs into a shared document by another means still conforms, as long
as clients observing the document see the outputs.

### Authorization

Requests are authenticated (standard Jupyter Server authentication) and
authorized under the `executions` auth resource. This allows an administrator to
control server-side execution as a distinct capability — for example, granting
read/write on documents and kernels while still restricting who may trigger
server-driven execution.

### Interaction with existing APIs

- This endpoint **does not** replace the kernel WebSocket protocol or the
  existing `/api/kernels` lifecycle endpoints. Client-driven execution over the
  kernel WebSocket continues to work unchanged.
- It **complements** the collaboration/document APIs: `document_id` refers to a
  document those APIs manage.
- It **does not** start, restart, interrupt, or delete kernels; those remain the
  province of the existing kernel APIs.

## Rationale and alternatives

**Why fire-and-forget instead of returning outputs in the HTTP response?**
Because in the server-side/collaborative model the document is already the
delivery channel to *all* clients, including the requester. Returning outputs in
the response would (a) duplicate that channel, (b) force the requester to merge
two sources of truth, and (c) make long-running or streaming cells awkward over
a single HTTP response. Decoupling the trigger from the results is what makes the
model robust to reconnects and to non-notebook clients.

**Why reference cells by ID + hash instead of sending source inline?**
The authoritative source lives in the shared document. Sending source inline
would reintroduce the possibility of the kernel running something different from
what the document holds, and would fight with concurrent edits. Referencing by
ID keeps the document as the single source of truth; the hash makes drift
explicit and safe (`409`) rather than silent.

**Why a non-cryptographic hash (MurmurHash2)?** The hash only needs to answer
"same source or not" cheaply. A cryptographic digest (e.g. SHA-256) would also
work and is more universally available across languages; MurmurHash2 was chosen
to match the reference implementation and its frontend. The trade-off is
recorded in [Unresolved questions](#unresolved-questions); the important
invariant is that client and server agree.

**Why explicit `request_id`/`previous_request_id` ordering instead of relying on
HTTP call order?** HTTP requests can arrive out of order, and a client may not
want to await each response before firing the next. Threading request IDs lets
the server reconstruct intent order without the client serializing its calls.
For the common case (one batch = one "Run All"), no IDs are needed at all.

**Why not extend the existing kernel WebSocket protocol instead of adding REST?**
A WebSocket requires a persistent, client-owned connection and client-side
routing — exactly the coupling this proposal removes. A stateless REST trigger
composes naturally with server-owned documents, works for headless/automation
clients, and is trivial to call from any language. The two coexist.

**Impact of not doing this.** Every project implementing server-side execution
continues to re-derive an incompatible variant of the same endpoint, frontends
and servers remain non-interoperable, and the pattern never lands in Jupyter
Server core where it could be shared.

## Prior art

- **jupyverse** ([#191](https://github.com/jupyter-server/jupyverse/pull/191))
  implements server-side execution with the fire-and-forget `null`-returning
  contract this JEP codifies; jupyverse will be updated with minor adjustments to
  conform.
- **jupyter-server-nbmodel** provides a server extension that executes cells
  server-side and writes outputs back through the document model.
- **jupyter-collaboration** ships `NotebookCellServerExecutor`, a client that
  speaks this exact request shape; it works unmodified against the reference
  implementation.
- **jupyter-server-documents**
  ([#248](https://github.com/jupyter-ai-contrib/jupyter-server-documents/pull/248))
  is the reference implementation from which this spec is drawn. Notably, that PR
  *removed* a large amount of custom kernel machinery (custom kernel client with
  a listener API, a message-cache mapping `msg_id`→`cell_id`, custom kernel
  managers, an iopub dispatch layer) in favor of this simpler REST + document
  model — direct evidence that the shared pattern reduces complexity.
- **IPython/Jupyter kernel protocol.** The `execute_request`/`iopub` model is the
  foundation; this proposal changes *who* drives it (server vs. client) and *how
  results are delivered* (document vs. WebSocket to the driver), not the kernel
  protocol itself.

The consistent lesson from these projects is that the pattern works and that its
main risk is fragmentation — which specification directly addresses.

## Unresolved questions

- **Hash algorithm portability.** v1 mandates MurmurHash2 (seed 0, decimal
  string) to match the reference implementation, but a cryptographic hash (e.g.
  SHA-256) would be more portable across languages. This should be settled
  before/while landing in Jupyter Server core, and the resolution recorded in the
  canonical OpenAPI spec. It is a candidate for change in a later API version.
- **`document_id` format.** The reference uses collaboration room names such as
  `json:notebook:<file_id>`. Whether the spec should mandate that exact format,
  or treat `document_id` as an opaque server-defined identifier, is open.
- **Non-notebook documents.** v1 targets notebook documents. Whether/how the API
  generalizes to other executable document types is out of scope here.
- **Relationship to sessions.** The endpoint keys off `kernel_id`; how it should
  interact with the Sessions API (which binds documents to kernels) may warrant
  clarification.

Explicitly **out of scope** for this JEP (deferred to future work, see below):
cancel/interrupt of a queued or running request, querying request/execution
status, and any endpoint beyond `execute`.

## Future possibilities

- **The OpenAPI spec becomes the living source of truth.** Once this API is
  added to Jupyter Server, its canonical definition lives as an OpenAPI
  specification in the
  [`jupyter-server/jupyter_server`](https://github.com/jupyter-server/jupyter_server)
  repository. This JEP defines and justifies **v1**; subsequent refinements
  (field additions, the hash decision, new response codes) can proceed through
  normal PR review against that OpenAPI spec without necessarily requiring a new
  JEP. This document should be read as the rationale for the first version, not
  as a frozen contract.
- **Additional lifecycle endpoints.** The natural extensions are a way to
  **cancel** a queued/running request, to **interrupt** execution, and to
  **query status** of a request. These were intentionally left out of v1 to keep
  the first version tightly scoped to what the ecosystem has already converged
  on; they can be added incrementally as implementations converge on their
  shapes.
- **Adoption in more frontends and servers.** With a single contract, JupyterLab,
  Notebook 7, jupyverse, and custom/automation clients can all rely on the same
  endpoint, and server-side execution can become a default capability of Jupyter
  Server rather than a per-project extension.
- **Agent and automation use cases.** A stateless, language-agnostic REST trigger
  for execution is a natural building block for headless and AI-agent-driven
  workflows that operate on live, collaboratively-viewed documents.
