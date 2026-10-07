# Quiet Ascent — V2 Book Commitments

A tamper-evident, timestamped record of what a systematic equity book held, published
**before** the outcome of each month was known. This is the second specification of the
strategy. The first has its own repository and its own chain; the two are linked, not merged.

## Why this exists

Published performance proves very little on its own. The only thing that makes a forward record
credible to someone who does not trust the author is a commitment made **before** the outcome is
known and **provably not altered afterwards**. Each month this repository records a cryptographic
fingerprint of the book, anchors it to a time nobody controls, and links it to the previous month
so the sequence cannot be rewritten.

**It is not a performance claim.** Nothing here asserts that this book made money.

## How V2 relates to V1

- **Same data, provably.** Both books are built from one vendor pull each month. Both repositories
  commit the identical `attestations/attestation_YYYY-MM.json`; the `body_sha256` inside it appears
  in both months' manifests.
- **Separate chain.** V2 has its own seed, anchor (`commit_anchor.json`), ledger and key chain. A key
  released here opens nothing in the V1 repository and vice versa.
- **Cross-linked.** Every V2 manifest records the V1 repository HEAD and V1's chain value for the
  same month under `lineage.v1_cross_link`.
- **The account.** The account that executed the V1 record through 2026-08 converted to the V2 book
  at the 2026-09-30 close. From 2026-09 the V1 record continues on paper and its manifests say so.

## What is committed, and when

Through 2026-09, one row: the traded book, `unrestricted.fully_invested`. From 2026-10, twelve rows,
the V1 shape: the unrestricted, core and capacity books, each under the four treatments
(fully invested, vol-managed, hedged, stacked), with the traded row unchanged (`public_terms.json`,
`roster.roster_changes`, dated 2026-09-30). The hedged and stacked rows are `hedged_conc` and `stacked_conc`:
a concentration-scaled RSP/IWM short, sized each month to the book's top-sector weight and blended to its own
cap profile (`roster_changes[1]`, dated 2026-10-07, before any twelve-row month was sealed; the rule and its
constants are in `roster.rows_note`). Every row is sealed on its own: `sealed/<month>/<row id>.seal.json`,
twelve envelopes a month from 2026-10, each named with its ciphertext hash in the manifest's `roster`.
A row is committed every month, run or not; a month that failed is still committed with `status: not-run`
and a stated reason. The licensed tiers are also published in the V2 documents, whose fingerprints are
in every manifest's `lineage.documents_sha256`; for 2026-09 their books were pinned only indirectly, through
the attested inputs and the engine hash in the lineage. Each manifest also carries the hashes of the
configuration, the engine and the builders the month's book came from, and the append-only register's
hash and byte length. Manifests from 2026-10 carry `schema: 2` for the twelve-row body.

## Sealed books: the anchor and the key chain

`commit_anchor.json` holds `K_0`, published once before any month was sealed. Keys form a reverse
hash chain, `K_n = SHA256(K_(n+1))`, and month *m* is sealed under `K_m`. Releasing `K_m` yields every
earlier key and no later one, so disclosure only moves forward and cannot skip a month. The rule is
in `public_terms.json`: the key for month *m* is released with the commitment of month *m+19*.

## Verify it yourself

```python
import hashlib, json
led = [json.loads(l) for l in open("book_commitments.jsonl") if l.strip()]
prev = hashlib.sha256(b"quiet-ascent-v2-books genesis").hexdigest()
for r in led:
    keys = ("protocol","schema","chain_index","month","stamp_date","sigdig","roster",
            "primitives","book_files","lineage","missing") + (
            ("not_run_acknowledgement",) if "not_run_acknowledgement" in r else ())
    body = {k: r[k] for k in keys}
    bs = hashlib.sha256(json.dumps(body, sort_keys=True, separators=(",",":")).encode()).hexdigest()
    assert bs == r["body_sha"] and r["prev_manifest_sha256"] == prev
    assert hashlib.sha256((prev + bs).encode()).hexdigest() == r["chain"]
    prev = r["chain"]
print("chain verified through", led[-1]["month"])
```

```python
# the register is append-only: truncate today's file to the recorded length and hash it
m = json.load(open("manifests/manifest_2026-09.json"))["lineage"]["pre_registration_v2"]
raw = open("PRE_REGISTRATION.md", "rb").read()
assert hashlib.sha256(raw[:m["bytes"]]).hexdigest() == m["sha256"]
```

```python
# same data as V1: the attestation committed here is byte-identical to V1's
a = json.load(open("attestations/attestation_2026-09.json"))["body_sha256"]
assert a == json.load(open("manifests/manifest_2026-09.json"))["lineage"]["attestation"]["body_sha256"]
```

Timestamps are OpenTimestamps proofs (`manifests/*.ots`), commits are signed with the key in
`allowed_signers`.
