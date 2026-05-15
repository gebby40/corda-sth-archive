# Corda STH archive

Public, append-only archive of Signed Tree Heads (STHs) issued by the
Corda Merkle audit log (cordalog).

## What's in here

- `latest.json` — the most recently published STH.
- `sths/<zero-padded-tree-size>.json` — every STH the operator has
  signed and published, one file per leaf count.

Each file is a verbatim copy of cordalog's `GET /log/sth` response
at the time of publication. No client-side reformatting.

## Why this repo exists

A malicious operator could show different STHs to different observers
(a "split-view attack"). Publishing every STH to a public, append-only
location makes the operator commit to a single timeline. Anyone can
clone this repo, compare it against what their local cordalog claims
is current, and detect equivocation.

## Verify

The operator's Ed25519 public key for STH signing is:

```
0c1624119b10d049a39216f7739b4403fdead060c9ee3116fbbec4e96aeed168
```

To verify a saved STH:

```sh
# fetch the audit tool from corda/log
cd corda/log
go run ./cmd/cordalog-audit consistency \
    --old sths/00000000000000000001.json \
    --new latest.json \
    --url https://log.corda.app \
    --pubkey 0c1624119b10d049a39216f7739b4403fdead060c9ee3116fbbec4e96aeed168
```

A passing run is a cryptographic proof that the log is a strict
extension of the older state — no leaves removed, no history rewritten.

## How it's produced

`cordalog-publish` runs every 5 minutes on the operator's host. It:

1. Fetches the latest STH from cordalog.
2. Verifies the STH signature under the pinned pubkey above.
3. Writes the STH to `sths/<size>.json` and updates `latest.json`.
4. Commits and pushes here.

If the publisher finds a different STH already archived at the same
`tree_size`, it refuses to overwrite — that condition is itself
evidence of operator equivocation. The deploy-key writes are
write-only (no force-push, no rewrite), so a malicious operator can't
silently amend history here either.

## Reporting equivocation

If you observe a divergence between this archive and the live cordalog,
open an issue with both STHs attached and the exact `--url` you used.
