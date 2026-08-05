# Webemoji Log (staging)

> **Staging deployment.** Own signing key, reset to genesis weekly, ephemeral —
> read [Staging notes](#staging-notes) at the bottom before verifying.

Public, append-only transparency log for Webemoji counters. This repository
holds the signed checkpoints, Bitcoin timestamps, Sigstore Rekor anchors,
Software Heritage archival records, signed daily statistics, and the raw log
entries themselves. Used with the open-source
verifier, it lets anyone recompute the counters and confirm the signed history
was not silently rewritten — a plain `git clone` of this repository is a
complete, offline-verifiable copy of the log.

This log is paired with the open-source
[`web-reactions-verifier`](https://github.com/khasky/web-reactions-verifier). This
repository holds the published data; that repository holds the code that checks it.

If you only want to check the current public log, start with **Verify** below.

## How verification works

Webemoji serves raw log entries from the public API (`/log/entries`),
mirrors them into this repository, and publishes signed tree heads here:

1. Each accepted counter-changing event is serialized as a log leaf.
2. The API periodically builds a Merkle tree over the leaves and signs the root
   as a checkpoint with Ed25519.
3. This repository records those checkpoints in Git history and mirrors the
   checkpoint-covered raw entries as `entries/` shards; mature checkpoints are
   also anchored to Bitcoin through OpenTimestamps and to Sigstore Rekor.
4. The verifier refetches the leaves (from the API or from the shards here),
   recomputes every leaf hash and the Merkle root, checks the signed checkpoint
   and the whole checkpoint archive, then folds the log back into counters.

That means live counters are verifiable against the public log. A cached or
served counter that does not match the fold of the signed log is detectable.

## What's here

Everything under `checkpoints/`, `ots/`, `entries/`, `rekor/`, `swh/`, and
`stats/` is written by the anchoring bot. What each file is for:

**`checkpoints/` — signed tree heads (STHs)**

- `latest.json` — the single newest checkpoint (`tree_size`, `root_hash`, `ts`,
  `signature`), pretty-printed. **Overwritten every checkpoint.** The O(1) entry
  point consumers and the verifier read first; also the git-published view that is
  compared against the live API to catch a split view.
- `<YYYY-MM-DD>.ndjson` — the permanent **append-only archive**: one compact JSON
  line per checkpoint, one shard per UTC day — the browsable history of every STH
  ever signed. Today's shard is appended to and freezes once the day rolls over.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

The newest checkpoint is in both `latest.json` and the current shard on purpose (a
moving pointer plus an append-only archive), not by accident.

**`ots/` — Bitcoin timestamps (OpenTimestamps)**

A checkpoint's root is submitted to the OTS calendars, then matured into a proof
once it lands in a Bitcoin block, so an anchored checkpoint `<tree_size>` produces:

- `<tree_size>.pending.json` — interim calendar receipt right after submission
  (queued, not yet in a Bitcoin block); superseded once the proof matures.
- `<tree_size>.ots` — the matured OpenTimestamps proof (standard binary `.ots`)
  anchoring that checkpoint's root in a Bitcoin block.
- `<tree_size>.json` — self-contained sidecar for that proof: the signed STH + the
  Bitcoin block height, so a verifier needs nothing else to tie the `.ots` to a
  checkpoint. (The block height lives here, not in the binary `.ots`.)
- `latest.json` — pointer to the newest matured proof; overwritten as proofs mature.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

Not every checkpoint gets its own OTS proof — only the newest not-yet-submitted one
each time submit runs; the rest ride a consistency proof to an anchored one.

**`entries/` — the raw log leaves, mirrored**

- `<start>-<end>.ndjson` — raw log entries in fixed 10 000-leaf ranges
  (zero-padded, e.g. `000000000001-000000010000.ndjson`), published once a
  checkpoint covers them — so the shards hold every leaf up to the latest
  checkpoint and nothing newer; a just-cast reaction appears here only after the
  next checkpoint seals it. One JSON line per leaf, byte-for-byte the same object
  the public API serves at `/log/entries`. A closed range is immutable; only the
  newest one grows.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

Because the leaves are mirrored here, a clone of this repository is a complete,
independently archivable copy of the log, and the verifier can audit it **fully
offline** (see Verify below) — the API being unavailable, or serving something
different, changes nothing about what this record proves.

Each leaf is pseudonymous by design. The `user_ref` field is a rotating
per-epoch pseudonym — not your account, email, or any stable identifier. It
changes every epoch and cannot be linked across epochs or back to a person, so
mirroring the full log here exposes activity, never identities.

Revocations are part of the same log: account erasure and other public
corrections are append-only `op=4` leaves, exposed at `/log/revocations` and
present in the shards. The verifier checks that endpoint against the actual
`op=4` leaves covered by the signed root anchored here.

**`rekor/` — Sigstore Rekor anchors**

- `<tree_size>.json` — sidecar for a checkpoint anchored to
  [Sigstore Rekor](https://docs.sigstore.dev/logging/overview/), an independently
  operated public transparency log: `{tree_size, root_hash, rekor_uuid,
  rekor_log_index, rekor_url}`. The submitted entry carries the same signed tree
  head bytes, so Rekor independently witnesses each checkpoint's existence; the
  verifier cross-checks this by default (`--no-rekor` to skip), resolving the
  UUID and comparing the bytes.

**`swh/` — Software Heritage archival records**

- `latest.json` — the coordinate of the most recent daily Software Heritage save
  attempt for this repo: `{origin, git_commit, swhid, revision_url,
  save_request_id, save_request_status, save_task_status, snapshot_swhid,
  requested_ts}`. **Rewritten once per day** whether or not that day's save
  request was accepted (the `save_request_*` / `snapshot_swhid` fields are null
  when it was throttled, but `git_commit` / `swhid` always pin the day's head);
  its git history here is the append-only record of every daily save attempt.

Because this is a Git origin, the archived commit's SWHID is
`swh:1:rev:<git_commit>` — SWHIDs are `sha1_git`, so the SWH revision hash *is*
the git commit hash. That makes third-party preservation checkable rather than
assumed: resolve `revision_url` (or the SWHID) against the Software Heritage API
and confirm the archive holds this repo's history. The save runs asynchronously,
so a just-published coordinate resolves once SWH completes the visit — the same
"pending, then matures" shape as an OTS proof.

**`stats/` — signed daily aggregates**

- `<YYYY-MM-DD>.json` — one signed commitment per UTC day: `votes`,
  `unique_user_refs`, and `revokes` (all three recomputable from the entries by
  anyone — the verifier does exactly that), `new_accounts` (an operator
  commitment not derivable from the log), and, on the day a pseudonym epoch
  closes, an `epoch_continuity` count. The Ed25519 signature covers a canonical
  text rendering and uses the same log key as the checkpoints. The day series is
  gap-free from its first file; the verifier flags holes and stale series.

## Reading the commit history

Every commit here is made by the anchoring bot. The emoji prefix says which
stage of the pipeline the commit belongs to, so a glance down the history
separates proofs still waiting on Bitcoin (⏳) from proofs already anchored (⚓):

| Prefix | Stage |
| --- | --- |
| 🌱 | raw leaves appended |
| 🔏 | signed artifact published |
| 🧭 | a pointer file moved |
| ⏳ | timestamp proof requested, not yet confirmed |
| ⚓ | timestamp proof confirmed |
| 📚 | third-party mirror refreshed |

The text after the prefix says what it did:

| Commit message | File written | What it means |
| --- | --- | --- |
| `🔏 add checkpoint 766` | `checkpoints/<YYYY-MM-DD>.ndjson` | a new Ed25519-signed tree head (STH) for `tree_size` 766 was appended — the substantive "a checkpoint was published" event |
| `🧭 update latest 766` | `checkpoints/latest.json` | the pointer to the newest checkpoint moved to 766 (the file the verifier reads) |
| `⏳ ots submit 759` | `ots/759.pending.json` | checkpoint 759's root was submitted to the OpenTimestamps calendars; awaiting a Bitcoin block |
| `⚓ ots anchor 759` | `ots/759.ots` | the proof matured — 759's root is now anchored in Bitcoin (the block height is recorded in the `ots/759.json` sidecar) |
| `⚓ ots sidecar 759` | `ots/759.json` | the self-contained sidecar for that proof (signed STH + block height) |
| `🧭 ots latest 759` | `ots/latest.json` | the pointer to the newest matured proof moved to 759 |
| `🌱 add entries 741-766` | `entries/<start>-<end>.ndjson` | leaves 741–766 (now covered by a checkpoint) were appended to the raw-entry shard |
| `⚓ rekor anchor 766` | `rekor/766.json` | checkpoint 766's signed tree head was submitted to Sigstore Rekor; the sidecar records the entry UUID |
| `📚 swh save a1b2c3d` | `swh/latest.json` | Software Heritage was asked to re-archive the repo; the record pins the archived commit `a1b2c3d` as `swh:1:rev:…` |
| `🔏 stats 2026-07-18` | `stats/2026-07-18.json` | the signed daily aggregates for that UTC day were published |

`tree_size` is the cumulative number of log leaves — it only ever grows.

**Commit messages are informational only.** The verifier never reads them: it
recomputes everything from the file *contents* here plus the public API's
`/log/*` endpoints. Read them to follow what the bot did; nothing depends on
their wording — including the prefixes, which older commits (published before
they were introduced) simply don't carry.

**Why the numbers look out of order.** Checkpoint `tree_size` values jump by
however many events landed in that hour (e.g. `742 → 754 → 759 → 766`), not by
one. And an OTS submit always anchors the *newest* checkpoint not yet submitted
(submits run right after each checkpoint), so several submits can walk newest →
older (`766`, then `759`, …). Both are expected; most intermediate checkpoints
never get their own OTS proof and are tied to an anchored one by consistency
proofs instead.

**Editing this repository.** Docs (`README`, `LICENSE`, anything outside the data
directories) are safe to edit — the verifier ignores them and the bot never
touches them. The data directories — `checkpoints/`, `ots/`, `entries/`,
`rekor/`, `swh/`, and `stats/` — are machine-generated: hand-editing them, force-pushing,
or rewriting history is exactly the tampering the verifier is built to catch (and
third-party mirrors preserve the real history). Don't edit them by hand.

## Verify

Set the coordinates of the deployment you are checking — every command below
reuses them:

```bash
API=https://api-staging.webemoji.app
REPO=https://raw.githubusercontent.com/khasky/web-reactions-log-staging/main
KEY="--pubkey <staging-ed25519-key-base64>"   # published with the staging deployment
```

### Fast check

Use the open-source verifier:

```bash
npx webemoji-verify --api $API --repo $REPO $KEY
```

To also compare one live target's `/reactions/count` response to the recomputed
fold, add `--target site/id`:

```bash
npx webemoji-verify --api $API --repo $REPO $KEY --target github/1
```

### Fully offline audit

The raw leaves are mirrored in this repository, so the whole audit can run
against a clone or mirror without contacting the API at all — the checkpoint
under test comes from `checkpoints/latest.json` and every entry from the
`entries/` shards:

```bash
npx webemoji-verify --entries repo --repo $REPO $KEY
```

### Bitcoin anchor check

`--ots` additionally verifies the newest matured OpenTimestamps proof against a
Bitcoin block. It is slower and can only pass after an OTS proof has matured, so
it is separate from the fast check:

```bash
npx webemoji-verify --api $API --repo $REPO $KEY --ots
```

### From a checkout

The verifier lives in a separate public repository:

```
git clone https://github.com/khasky/web-reactions-verifier
cd web-reactions-verifier
pnpm install
node src/verify.mjs --api $API --repo $REPO $KEY
```

The production public key lives in one authoritative place — pinned in the
[verifier source](https://github.com/khasky/web-reactions-verifier/blob/main/src/verify.mjs)
(and printed in that repository's README) — deliberately not restated here, so a
copy can't silently drift from the one the tool actually checks against. Any
other deployment — staging, a fork, a different signing key — is verified by
passing `--pubkey <base64>`; that is what `KEY` above carries.

With `--target github/1`, expected successful output looks like this. Without
`--target`, the live `/reactions/count` comparison line is omitted.

```bash
checkpoint: tree_size=1128 ts=1784586730193
PASS  checkpoint Ed25519 signature
PASS  checkpoint is fresh (2.1h old, threshold 168h — a quiet log ages legitimately; tune --max-checkpoint-age-hours)
PASS  GitHub anchor matches signed root (tree_size=1128)
PASS  fetched all 1128 leaves (got 1128, source: api)
PASS  every recomputed leaf_hash matches the served leaf (0 mismatch)
PASS  recomputed Merkle root == checkpoint root_hash
PASS  checkpoint archive parses (45 STH line(s) in 13 shard(s))
PASS  no two archived STHs disagree on one tree_size (0 conflict(s))
PASS  every archived STH signature verifies (45 checked, 0 bad)
PASS  archived STH timestamps are monotone in tree_size (0 regression(s))
PASS  archive never exceeds the live tree (max archived 1128 <= 1128)
PASS  the live checkpoint is present in the archive shards
PASS  every archived root replays from today's leaves (45 checkpoint(s), 0 mismatch)
PASS  signed daily stats match the log (11 day(s), 0 violation(s))
folded 793 (site,target,reaction) counters from 1128 events
PASS  live /reactions/count matches the fold for github/1
revocations: 308 tombstone(s)
PASS  /log/revocations matches op=4 leaves in the log (308)
PASS  structural invariants hold (0 violation(s))
PASS  account wipes are complete (0 violation(s); grace 48h)

RESULT: PASS
```

Exit code `0` means the verifier passed. Exit code `1` means the public entries,
signed checkpoint, GitHub anchor, revocation list, or live counters did not match
what the verifier recomputed.

## Reactions, removals, and tombstones

The log records counter-changing events, not just final state:

- `op=1` — a reaction was added.
- `op=2` — a reaction was changed; the leaf records both the new reaction and
  the previous one.
- `op=3` — a reaction was removed by the user.
- `op=4` — a revocation tombstone: a later public leaf that reverses an earlier
  `op=1`, `op=2`, or `op=3` leaf.

So a normal user "unreact" is `op=3`, not a tombstone. Tombstones are for
append-only corrections such as account erasure or other public reversals. The
original leaf stays in the log; the `op=4` leaf points at it with `revoke_seq`,
and the verifier applies the inverse effect when recomputing counters.

## What this proves

The verifier checks integrity of the public counter history:

- the checkpoint signature matches the published Ed25519 key;
- the entries (from the API or the shards here) recompute to the signed Merkle root;
- the root matches the checkpoint published in this repository;
- every checkpoint ever archived here replays from today's leaves — the whole
  published history lies on one append-only line;
- the signed daily stats match what the log itself contains for each day;
- the revocation endpoint matches the actual `op=4` leaves in the log;
- optional target counts match the fold of the signed log;
- the checkpoint's Sigstore Rekor entry holds exactly its signed bytes (checked by default; `--no-rekor` to skip);
- with `--ots`, a matured checkpoint root is anchored in Bitcoin.

This does **not** prove that every reaction came from a unique human, or that the
anti-abuse policy is perfect. It proves that the published counters match the
public append-only log and that changes/removals/revocations are represented as
verifiable log events.

## Administrators

This repository holds only the published log data and its documentation — no
scripts or tooling. Operator maintenance (resetting the log to genesis,
scaffolding a fresh empty layout) is performed by the backend automation and
lands here as normal bot commits, visible in the commit history like everything
else.

Force-pushing or rewriting history in this repo is itself the tamper signal —
third-party mirrors (e.g. Software Heritage) preserve the real history.

## Staging notes

Everything above describes this repository exactly as it describes the
production log — the two READMEs are kept identical. What differs is only this:
this repository is the log of the Webemoji **staging** deployment at
`https://api-staging.webemoji.app`, not the production log at
[web-reactions-log](https://github.com/khasky/web-reactions-log).

- **Own signing key.** Staging checkpoints are signed with a separate Ed25519
  key, which the verifier does not pin — it pins the production key. Every
  verification run therefore needs `--pubkey`, which is what the `KEY` variable
  at the top of **Verify** carries. The production log needs no `--pubkey`.
- **Reset weekly.** This log, and the entire staging database behind it, is
  **fully reset to genesis every Monday (around 03:00 UTC)**, and on demand. So
  `tree_size` only ever grows *between* resets, and the daily `stats/` series
  restarts with each one.
- **Ephemeral by design.** Any checkpoint, proof, or entry here is wiped at the
  next reset and the log is rebuilt from scratch. Nothing in this repository is
  durable enough to cite or archive against — for that, use the production log.

## License

The log data in this repository is dedicated to the public domain under
[CC0 1.0 Universal](LICENSE) — copy, mirror, and verify it freely.
