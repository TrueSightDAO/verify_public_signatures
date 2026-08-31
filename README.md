# Verify Public Signatures

**The public, auditable RSA attestation ledger for TrueSight DAO.**

Every RSA-signed event submitted to the DAO (via Edgar / the farmer apps) that passes
verification is published here as one **immutable JSON file per event**, organized by
event type. Anyone — a verifier, a VVB, a buyer, a partner — can independently re-verify
any attestation offline using only the standard `openssl` tool. No trusted intermediary
required.

## Why per-event files (not one big aggregate)

- **No size ceiling** — each event is one small, immutable file; the ledger grows
  linearly without ever forcing a full rewrite.
- **Real per-attestation URLs** — every event has a stable URL that can be cited
  (e.g. in a future `[CARBON CREDIT ISSUANCE EVENT]`).
- **Append-only emission** — writers (the reconciliation cron today, dao_protocol's
  emit hook later) add one file per event; no read-modify-write, no races.
- **Git history is the audit trail** — each publish is a commit.

## Layout

```
verify_public_signatures/
├── README.md
├── index.json                       # root index: totals + per-event-type pointers
├── tree_planting/
│   ├── index.json                   # folder index: id -> url + metadata
│   └── <telegram_message_id>.json   # one file per event (immutable once written)
├── tree_planting_link/
│   ├── index.json
│   └── <message_id>.json
├── tree_planting_reject/
│   ├── index.json
│   └── <message_id>.json
├── tree_growth_monitoring/
│   ├── index.json
│   └── <message_id>.json
└── contribution/        # future event types land in their own folders
├── sales/
└── inventory_movement/
```

## Event record schema (per-event file)

Each `<message_id>.json` carries the full self-verifying triple:

| Field | Description |
|---|---|
| `event_type` | e.g. `[TREE PLANTING EVENT]`, `[TREE GROWTH MONITORING EVENT]` |
| `telegram_message_id` | Stable dedup key (Telegram message ID) |
| `telegram_update_id` | Telegram update ID |
| `submitted_at` | Submission date |
| `contributor_name` | Display name of the signer |
| `public_key` | **RSA-2048 public key (SPKI, base64)** — the signer's key |
| `signature` | **RSASSA-PKCS1-v1_5 signature (base64)** over `signed_payload` |
| `signed_payload` | **The exact bytes that were signed** (text up to and including the `--------` separator, stripped) |
| `signed_text` | Full original submission text (context) |
| `source_tab` | Source sheet tab |
| `verifiable` | `true` if the signature verified against the key at publish time |
| `linked_tree_id` | Linked tree ID when applicable |

To re-verify: sign `signed_payload` with `public_key` using RSASSA-PKCS1-v1_5 + SHA-256,
and compare with `signature`. Concrete command below.

## Index schemas

**Root `index.json`** — aggregates by event type:

```json
{
  "status": "success",
  "schema_version": 1,
  "generated_at": "2026-08-31T17:19:56Z",
  "total_count": 74,
  "test_events_count": 25,
  "event_types": {
    "tree_planting": { "count": 24, "index_url": "…/tree_planting/index.json" },
    "tree_planting_link": { "count": 8, "index_url": "…" },
    "tree_planting_reject": { "count": 41, "index_url": "…" },
    "tree_growth_monitoring": { "count": 1, "index_url": "…" }
  }
}
```

**Folder `index.json`** — the enumeration surface for one event type:

```json
{
  "status": "success",
  "schema_version": 1,
  "generated_at": "…",
  "event_type": "[TREE PLANTING LINK EVENT]",
  "count": 8,
  "events": {
    "Edgar_20260820112723_046": {
      "url": "…/tree_planting_link/Edgar_20260820112723_046.json",
      "event_type": "[TREE PLANTING LINK EVENT]",
      "submitted_at": "2026-08-20",
      "contributor_name": "Edgar"
    }
  }
}
```

## Verify any signature (offline, openssl)

```bash
# 1. Fetch an event file
curl -sL https://raw.githubusercontent.com/TrueSightDAO/verify_public_signatures/main/tree_planting/171.json -o event.json

# 2. Reconstruct the PEM public key (the stored key is bare SPKI base64)
python3 - <<'EOF'
import json, base64
d = json.load(open("event.json"))
open("pub.pem", "w").write("-----BEGIN PUBLIC KEY-----\n" + d["public_key"] + "\n-----END PUBLIC KEY-----\n")
open("payload.txt", "w").write(d["signed_payload"])
open("sig.bin", "wb").write(base64.b64decode(d["signature"] + "=="))
EOF

# 3. Verify
openssl dgst -sha256 -verify pub.pem -signature sig.bin payload.txt
# => Verified OK
```

`Verified OK` proves the event was signed by the holder of the corresponding private key
and that the payload was not altered.

## Privacy

- **No PII.** A fail-closed scan rejects any record containing email-like patterns or
  other personal identifiers before publish.
- Public keys, display names, and already-public tree/geo data are included by design —
  they are the attestation, not secrets.
- Test/synthetic and malformed submissions are **bucketed separately** (`test_events_count`
  in the root index) and are never published as attestations.

## Refresh

The ledger is refreshed by a reconciliation cron on the autopilot box every 30 minutes
(`sync_sunmint_signatures.py`, same pattern as the proven `sync_pending_caches.py`).
It is incremental and sha-aware: only new/changed files are pushed, so historical
records are never rewritten. A dao_protocol emit hook is planned so new verified events
are published at ingest time; the cron remains as the reconciliation/backfill safety net.

## Future event types

Contribution reporting, sales, and inventory movement RSA events will land in their own
folders (`contribution/`, `sales/`, `inventory_movement/`) as the emit path widens.
