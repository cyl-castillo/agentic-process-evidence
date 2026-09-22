# Signed session logs (Testigo) - Overview

## Scenario

An operator runs an agent against production. One session: a human prompt, three agent commands, each individually approved by a human, no files changed.

The session log is not a plain timeline JSON: it is a **signed session-chain statement** (a [Testigo](https://github.com/cyl-castillo/testigo) proof packet) - an in-toto Statement in a DSSE envelope whose predicate carries the hash-chained events of the session.

Process ends at the **turn end**; the process is one session, so the subject of the process evidence is the session log itself.

## Whats the point of adding this

A session log that carries its own integrity and signature:

- Verifies **without access to the log store**. A gate, an auditor or a customer holding the bytes recomputes the chain and the signature in a browser.
- Supports **selective disclosure**. Events can be redacted before the log leaves the organisation; the receiver verifies linkage across the redaction and is told which events were withheld. Here 7 of 12 events are redacted, the three human approval decisions and the empty diff are fully verifiable.
- Records **human approval decisions with a stated reason**, hash-chained between the agent's request and the tool result - the per-action evidence for "human oversight" questions.
- Is **addressable per event**: alignment evidence or a reviewer can cite an approval or a tool call by content hash.

APE stays the process-level statement. The packet is one admitted shape of `sessionsLogs[]`: `uri + digest`, exactly as today. Plain timelines are unaffected.

## Mental Model

### Accountability

The operator who prompted, approved each command and exported the packet is the process owner and the reviewer. The packet is signed with their key; the key id is the trust anchor, compared out-of-band.

### Agentic session

One Claude Code session inside [agent-console](https://github.com/cyl-castillo/agent-console) (the Testigo reference implementation), which sits in the permission path and records approvals as events.

### Agentic process evidence

Its subject points to the session log (uri + sha256 of the packet file)
It carries `providers` (harness), `tools`, `owner`, `reviewers`, timestamps and, under `custom.testigo`, what the packet itself proves: signer key id, segment digest, redacted entries, approval decisions, turn diff

## Artifacts

- **testigo-session-log.proofpack.json** - the signed session log (real, from production, redacted). Verify it at https://cyl-castillo.github.io/testigo/verifier/testigo-verifier.html or with any DSSE tooling.
- **agentic-ops-process-evidence.json** - the process evidence on it (unsigned example, generated from the packet by [`examples/ape/build.mjs`](https://github.com/cyl-castillo/testigo/blob/main/examples/ape/build.mjs)).

## See also

- `[implementation_details.md](./implementation_details.md)`
- `[objects_examples/](./objects_examples/)`
- Field-by-field mapping APE ↔ Testigo: https://github.com/cyl-castillo/testigo/blob/main/docs/ape-mapping.md
