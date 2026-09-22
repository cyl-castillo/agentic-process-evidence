# Signed session logs - Implementation

Same two elements as the other use cases. What changes is the shape of the session log.

1. **Agent runtime tool** - captures the session into a local, append-only, hash-chained ledger; at process end the human reviews, redacts and signs the segment; the runtime tool uploads the packet and pushes the process evidence
2. **Server** - persists packet + evidence, verifies both, gates on them

---

## 1. Agent runtime tool

### Telemetry capture - the ledger

Every event is one JSONL line with `seq`, `ts`, `caseId`, `turnId`, `kind`, `actor`, `payload`, `prevHash`, `hash`. `hash` is sha256 over the line bytes with the hash emptied; `prevHash` is the previous line's hash. Cost: one sha256 per event. No signature until export.

Event kinds in this example: `prompt` (human), `snapshot` (working tree before the turn), `approval_request` (agent asks to run a tool), `approval_decision` (human: `allow|deny|ask` + reason), `tool_result`, `turn_end` (files changed, tree after). Testigo v0.2 adds `session_start` / `model_switch` (model identity) and `external_evidence` (a commitment to a platform-held record, e.g. the harness transcript or a Compliance API export - `uri + sha256`, the same shape as `sessionsLogs[]`).

### Aggregating - the case

A `caseId` groups the turns of one intent thread (`jira:KEY`, `github:org/repo#N`, or the terminal). Export selects a contiguous segment; events of other cases inside the range become stubs (hashes only), so linkage verifies without sharing them.

### Review, redaction, signing

Before anything is signed the human sees everything the packet would contain and marks events to redact. Redaction replaces the payload and keeps `seq`/`prevHash`/`hash`; `redactionCount` in the predicate must equal the redacted entries. The segment is then wrapped as an in-toto Statement (`predicateType` `https://github.com/cyl-castillo/testigo/attestation/v0.1`), signed as DSSE ed25519, optionally timestamped (RFC 3161 over the signature).

The predicate also carries `provider` (APE `Provider` shape), `contextArtifacts` (APE shape, derived from instruction-file digests recorded at prompt time), `startTimestamp` / `endTimestamp` (checked by verifiers against the first and last event) and `owner`.

### Evidence push

Upload the packet (`uri` + `sha256` of the file). Craft the process evidence with the packet as `sessionsLogs[]` entry - or, as here, as `subject` when the process is one session - and push.

---

## 2. Server

### Persistence

Same as the other use cases. The packet is a self-contained file; retention is retention of bytes.

### Verification

Before trusting `sessionsLogs[]`, the server MAY verify the packet itself (the plain-timeline case has nothing to verify beyond the digest):

1. `keyid` = sha256 of the embedded public key; compare with the allow-listed operator keys
2. DSSE signature over the payload
3. subject digest = sha256 of the packed events
4. linkage: every `prevHash` equals the previous `hash`, from `range.prevHashBefore`
5. content hash recomputes for every non-redacted event; `redactionCount` equals the redacted entries
6. if present: process-context fields well-formed, `startTimestamp` / `endTimestamp` equal the first / last event

Reference verifiers: the browser page, a zero-dependency Node script, and a conformance corpus of 21 vectors (including valid signatures wrapped around internal defects) at https://github.com/cyl-castillo/testigo/tree/main/conformance.

### Gates

Everything the other use cases gate on, plus what only a verified packet can answer:

- Every tool call the agent made was approved by a named human (`approval_decision` per `approval_request`)
- The turn changed nothing (`turn_end.filesChanged: []`) - or exactly the files the ticket allowed
- Which events were withheld from this receiver, explicitly

---

## Artifacts

### testigo-session-log.proofpack.json

Real: a production deploy verification of Fixy (2026-07-15), re-exported with agent-console 0.79.0. 12 events, 3 human approvals, empty diff, 7 redacted entries, chain intact. File sha256 `2c9d7f01ea070530b6bb25084d3e274a197c3e4d14508cb69281ab732145b23b`; segment digest `af21a0b7bfe48022764831415f1c82f45211f0b886ad59bbe4a19842308e0ceb`; signer key id `8caf09075df11abbbdea5cd1d120a5654d8d1ce2a32e2a6f018dde557c5014da`.

### agentic-ops-process-evidence.json

- `predicateType`: `https://jfrog.com/evidence/agentic-dev-process/v1` (reused; an ops-specific type would fit better)
- `subject` = the packet file (uri + sha256)
- `sessionsLogs[]` omitted - the subject is the log
- `providers` = harness from the packet's `provider`
- `tools` = distinct tools in the events
- `custom.testigo` = signer key id, segment digest, ledger head, redacted entries, approval count, turn diff
- `owner` / `reviewers` = the operator
- `startTimestamp` / `endTimestamp` = from the packet
- unsigned on purpose: it shows the shape, it does not claim provenance for itself
