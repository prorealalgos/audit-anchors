# ProRealAlgos audit attestations

This repository is a public, append-only ledger of cryptographic attestations
for every trade ProRealAlgos has published on its live audit trail at
[prorealalgos.com/AuditTrail](https://prorealalgos.com/AuditTrail).

**Per-close model.** As soon as a trade closes and propagates into the audit
trail cache, an automated job commits `anchors/T-XXXXXX.json` to this repo.
Each anchor file is self-contained: it carries the canonical receipt JSON,
the SHA-256 of that JSON, and the server-side timestamp at the moment of
commit. The commit itself is timestamped by GitHub.

**Why this matters.** GitHub controls the commit timestamps, the history is
public, and a force-push that rewrites history leaves a public reflog entry.
Once a trade's anchor file has been committed, ProRealAlgos cannot quietly
alter, delete, or back-date that trade without the change being externally
visible. The receipt JSON is embedded in the file, so a verifier doesn't need
to trust anything we publish on the live site — they can re-hash the embedded
receipt and confirm it matches.

## How to verify

```bash
# 1. Pull the anchor file for the trade.
curl https://raw.githubusercontent.com/prorealalgos/audit-anchors/main/anchors/T-XXXXXX.json > anchor.json

# 2. Extract the canonical receipt JSON and the claimed hash.
jq -c .receipt anchor.json > receipt.json
jq -r .hash anchor.json
# → a3f9c1b27d40e8927ac4f0e2b6c8d51f3a7e9c2b4d6f8a1c0e2b4d6f8a1c3e5b

# 3. Re-hash the receipt locally. The result must match step 2's claimed hash.
sha256sum receipt.json

# 4. Check GitHub's commit timestamp for the file.
git log -1 --format='%aI' anchors/T-XXXXXX.json
# → 2026-05-29T12:34:56Z
```

If steps 3 and 4 succeed, the trade had exactly those values at the timestamp
GitHub stamped on the commit. Anything else means the file was tampered with.

## Anchor file format

```jsonc
{
  "schema":         "prorealalgos.audit-attestation.v1",
  "trade_ref":      "T-XXXXXX",
  "close_ref":      "<IG close deal reference>",
  "attested_at":    "2026-05-29T12:34:56Z",
  "hash":           "<64-char hex sha256>",
  "hash_algorithm": "sha256",
  "leaf_format":    "sha256(canonical receipt JSON, UTF-8 bytes)",
  "receipt":        { /* the canonical receipt JSON, verbatim */ }
}
```

## What this repo is NOT

- This is **not** a substitute for IG's official broker statements. The trades
  are executed at IG; this repo proves we cannot retroactively rewrite the
  versions of those trades we publish.
- The `attested_at` timestamp in each file comes from the ProRealAlgos server
  at the moment of commit. The authoritative external timestamp is whatever
  GitHub stamps on the commit itself. They should be within seconds of each
  other; if they diverge meaningfully, trust the commit timestamp.
