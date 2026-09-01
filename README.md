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
├── <event_type_folder>/             # one folder per RSA-signed event type
│   ├── index.json                   # folder index: id -> url + metadata
│   └── <message_id>.json            # one file per event (immutable once written)
```

**Live event-type folders (33 as of the 2026-09 A6 backfill):**

| Folder | Events | Folder | Events |
|---|---|---|---|
| `contribution_event` | 2,172 | `notarization_event` | 38 |
| `inventory_movement` | 661 | `invoice_contribution` | 33 |
| `sales_event` | 282 | `tree_planting` | 27 |
| `dao_inventory_expense_event` | 201 | `proposal_vote` | 20 |
| `practice_event` | 170 | `qr_code_update_event` | 15 |
| `credentialing_attestation_event` | 109 | `asset_receipt_event` | 14 |
| `donation_mint_event` | 102 | `batch_qr_code_request` | 13 |
| `tree_planting_reject` | 42 | `proposal_creation` | 11 |
| `upc_linking_contribution` | 11 | `repackaging_settlement_event` | 3 |
| `tree_planting_link` | 9 | `qr_code_event` | 3 |
| `farm_registration` | 8 | `test_event` | 3 |
| `retail_field_report_event` | 8 | `currency_conversion_event` | 2 |
| `farm_boundary_evidence_event` | 6 | `tree_growth_monitoring` | 2 |
| `partner_check_in_event` | 6 | `voting_rights_withdrawal_settlement_event` | 2 |
| `capital_injection_event` | 5 | `email_verification_event` | 4 |
| `repackaging_batch_event` | 4 | `store_add_event` | 4 |
| `farm_registration_event` | 3 | | |

Counts drift as new events land; the root `index.json` is the live source of truth.
New event types appear automatically (folder = snake_case of the event-type marker).

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
  "total_count": 3947,
  "test_events_count": 25,
  "excluded_pii_count": 1650,
  "event_types": {
    "contribution_event": { "count": 2172, "index_url": "…/contribution_event/index.json" },
    "inventory_movement": { "count": 661, "index_url": "…" },
    "sales_event": { "count": 282, "index_url": "…" },
    "tree_planting": { "count": 27, "index_url": "…" }
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

- **Email-bearing events are excluded by policy (governor decision 2026-09-02).** Any RSA
  event whose signed text contains an email-like pattern is bucketed into
  `excluded_pii_events` and **never published**. The root index reports the count as
  `excluded_pii_count` (1,650 events excluded during the A6 backfill).
- **Fail-closed scan.** Every build pass re-scans the full published set for email-like
  patterns and refuses to push if any slip through. `--allow-pii` exists only as an
  explicit override and is **never used by the cron**.
- Public keys, display names, and already-public tree/geo data are included by design —
  they are the attestation, not secrets.
- Test/synthetic and malformed submissions are **bucketed separately** (`test_events_count`
  in the root index) and are never published as attestations.

## Refresh

- **Ingest-time emit (live):** dao_protocol publishes each verified RSA event immediately
  at verify time (`ledger_emit.emit()`), so new attestations appear within seconds.
- **Reconciliation cron (safety net + backfill):** the autopilot box runs
  `sync_sunmint_signatures.py --push` every 30 minutes. It is incremental, sha-aware
  (content-addressed skip — already-published files are never re-pushed), idempotent by
  message ID, and rate-limit-guarded (250 uploads/run + 0.3s inter-upload delay, with a
  cursor so each pass resumes where the last left off).
- **Historical backfill (2026-09):** all RSA-signed events in the Telegram Chat Logs are
  published to their event-type folders; email-bearing events are excluded per policy.
- Git history is the audit trail — each publish is a commit.

