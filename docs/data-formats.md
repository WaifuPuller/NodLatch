# Nodlatch data formats (design document)

Three small JSON files move between the laptop and the iQOO phone through Office Kit. They are kept single-line-safe (content blobs base64-encoded if they travel via the clipboard). The replacement file contents never travel to the phone; they stay in a laptop-side store keyed by SHA-256. The phone authorizes a hash; the executor checks the bytes against it.

## proposal.json  (laptop → phone)

```json
{
  "proposal_id": "P-001",
  "repo": "demo-repo",
  "created_at": "2026-09-26T12:40:00+05:30",
  "expires_at": "2026-09-26T12:50:00+05:30",
  "actions": [
    {
      "action_id": "A1",
      "type": "modify_file",
      "path": "src/auth.py",
      "expected_old_sha256": "9f2c…",
      "new_file_sha256": "b71e…",
      "summary": "Fix token expiry comparison in verify_token()",
      "risk": "low"
    },
    {
      "action_id": "A2",
      "type": "modify_file",
      "path": "config.py",
      "expected_old_sha256": "0a44…",
      "new_file_sha256": "c3d9…",
      "summary": "Change SESSION_TTL from 3600 to 86400",
      "risk": "medium"
    }
  ],
  "test_suites": [ { "id": "auth-suite-v1", "min_tests": 17 } ],
  "review_view": { "A1": "…redacted diff text…", "A2": "…" },
  "commitment_sha256": "e88a…"
}
```

`commitment_sha256` is the hash of the exact bytes of every field above except `review_view`, so the commitment graph stays acyclic.

## decision.json  (phone → laptop)

```json
{
  "decision_id": "D-001",
  "proposal_id": "P-001",
  "commitment_sha256": "e88a…",
  "approved": ["A1"],
  "denied": ["A2"],
  "required_test_suite": "auth-suite-v1",
  "signed_at": "2026-09-26T12:42:10+05:30",
  "signature": "base64 Ed25519 signature over the exact bytes of the fields above"
}
```

The phone signs the exact serialized bytes; the laptop verifies those same bytes and never re-serializes.

## receipt.json  (laptop → phone, informational)

```json
{
  "decision_id": "D-001",
  "status": "APPLIED",
  "applied": ["A1"],
  "denied": ["A2"],
  "tests": "17 passed",
  "finished_at": "2026-09-26T12:43:05+05:30"
}
```

## Status codes

`APPLIED` · `DENIED_BY_USER` · `TESTS_FAILED_NOT_APPLIED` · `REJECTED_MALFORMED` · `REJECTED_BAD_SIGNATURE` · `REJECTED_PROPOSAL_MISMATCH` · `REJECTED_EXPIRED` · `REJECTED_REPLAY` · `REJECTED_ZERO_ACTIONS` · `REJECTED_ARTIFACT_HASH_MISMATCH` · `REJECTED_SOURCE_STATE_MISMATCH` · `REJECTED_PROTECTED_TARGET` · `REJECTED_UNKNOWN_SUITE` · `REJECTED_UNSUPPORTED_ACTION` · `EXECUTION_ERROR_ROLLED_BACK` · `UNKNOWN_TRANSACTION_STATE`
