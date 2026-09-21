# Nodlatch — Phase 1 proposal (iQOO Hackathon 2026)

> **This repository contains planning material only: no code.**
> Nodlatch will be implemented in a separate repository created when the event clock starts on 26 September 2026, in line with the hackathon's original-work rule. Everything here is a design document.

**Human authorization for AI coding agents.**
*AI proposes. Human authorizes. Executor enforces.*

Team: Aaditya Singh (lead) · Sasi Chandan · Santhosh Kumar — Malla Reddy University
Track: Developer Tools · Hyderabad City Battle, 26–27 Sep 2026

---

## The problem

AI coding agents inspect repositories, edit files and iterate on their own. The human approval step is usually one **Allow** button. It rarely says which files, which bytes, or under what condition. The real question is not *did the user click Allow*, but **what exactly did the user authorize?**

## The idea

Nodlatch turns approval into an authorization artifact that constrains what executes.

```
AI agent proposes exact file changes as numbered actions  (A1, A2, …)
  → a deterministic builder records hashes and a review view
  → proposal.json crosses to the iQOO phone via Office Kit
  → the developer approves / denies each action and confirms a read-back
  → the phone signs decision.json (Ed25519 key never leaves the phone)
  → decision.json returns to the laptop via Office Kit
  → the executor verifies independently and applies ONLY approved actions
  → a trusted test suite gates the commit; original bytes kept for rollback
  → receipt.json goes back to the phone
```

**Example.** The agent proposes `A1 modify src/auth.py` and `A2 modify config.py`. On the phone the developer approves A1 and denies A2. The executor runs A1 only. A2 never executes because it was never authorized.

## Roles and trust

| Component | Role | Trust |
|---|---|---|
| AI coding agent (local model via Ollama, or another agent) | proposes changes | untrusted |
| Proposal builder (laptop) | derives paths, hashes, review view, redaction | trusted within the laptop |
| Nodlatch app (iQOO phone) | review, per-action decision, signing key | the authorization surface |
| Office Kit | moves three small JSON files between devices | transport only, never trusted |
| Executor (laptop) | verifies, applies, tests, rolls back | trusted enforcement point |

Deterministic code performs every security check. No model is in the enforcement path.

## What the executor checks, in order

1. Strict schema of `decision.json` (unknown fields → reject)
2. Ed25519 signature against the paired phone key
3. `proposal_id` and proposal commitment hash match the stored proposal
4. Not expired (laptop clock)
5. Not used before; mark used **before** the first write (single-use)
6. Approved / denied IDs exist; approved set non-empty
7. Stored replacement bytes hash to `new_file_sha256`
8. Current file on disk hashes to `expected_old_sha256`
9. Path is relative, no traversal, not in the protected list (`.env*`, keys, `tests/**`, `pytest.ini`, …)
10. Test suite resolved from the executor's own trusted config
11. Apply in staging, run the suite, require exit 0 **and** at least the expected number of tests
12. Re-check source, back up old bytes, write, verify, receipt

Any failure prints one status line (`REJECTED_ARTIFACT_HASH_MISMATCH`, `TESTS_FAILED_NOT_APPLIED`, `REJECTED_REPLAY`, …) and stops.

## Demo plan (four scenarios)

1. **Selective approval** — approve A1, deny A2 → `APPLIED (A1)`
2. **Tampered artifact** — one byte changed after approval → `REJECTED_ARTIFACT_HASH_MISMATCH`
3. **Failing test** — authorized change breaks the suite → `TESTS_FAILED_NOT_APPLIED`
4. **Replay** — same decision submitted twice → `REJECTED_REPLAY`

## MVP scope for the 30-hour build

One executable action type (`modify_file`, exact whole-file replacement); proposal and action IDs; per-action approve/deny on the phone; signed, single-use, time-limited decisions; executor verification with source-state recheck and protected paths; trusted test gate; exact-byte backup and rollback; the four demos. Build order and cut list are in the [Build Playbook](docs/Nodlatch_Build_Playbook.pdf).

## What Nodlatch is not

- Not an OS sandbox. An agent with the executor's OS privileges could bypass it. Nodlatch governs actions routed through the executor.
- A signature proves control of a key, not that a human was present. Phone-local user verification is planned, not claimed.
- Passing tests and matching hashes are conditions, not proof that code is safe.
- Redaction reduces exposure of secrets on the phone; it cannot find every secret.

## Contents

| File | What it is |
|---|---|
| `docs/Nodlatch_Phase1_Deck.pdf` / `.pptx` | 12-slide Phase 1 deck |
| `docs/Nodlatch_Build_Playbook.pdf` / `.html` | best-case demo flow, file formats, executor checks, seven-slice build order, hour-one device tests |
| `docs/data-formats.md` | example `proposal.json`, `decision.json`, `receipt.json` |
| `docs/Phase1_Form_Answers.md` | the Phase 1 form text as submitted |
| `media/Nodlatch_Walkthrough.mp4` | 1 min 52 s concept walkthrough (also at https://youtu.be/62Fikdp29bo) |
| `media/Nodlatch_Thumbnail.png` | video thumbnail |

## Planned open-source components (to be attributed in the build repo)

An open-source model runtime and model (e.g. Ollama with an open-weights model), an Ed25519 library, git, pytest, and a standard phone UI framework (Android / Flutter / PWA, decided after first-hour device tests).

---

*Prepared 19–21 September 2026 for the Phase 1 idea submission. No Nodlatch code exists yet.*
