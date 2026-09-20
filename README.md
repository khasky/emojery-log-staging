# Emojery Log (staging)

> **Staging deployment.** Own signing key, reset to genesis weekly, ephemeral. Read [Staging notes](#staging-notes) at the bottom.

Public, append-only transparency log for Emojery counters. This repository holds the signed checkpoints, Bitcoin timestamps, Sigstore Rekor anchors, Software Heritage archival records, and a manifest naming the raw log entries by digest. The entries themselves are served from object storage the manifest points at, so a `git clone` plus that one host is a complete, offline-verifiable copy of the log.

The code that checks it lives in the open-source [`emojery-verifier`](https://github.com/khasky/emojery-verifier): it recomputes the counters from the data published here and confirms the signed history was not silently changed.

If you only want to check the current public log, start with [Verify](#verify) below.

## How verification works

Emojery publishes the raw log entries where this repository commits to their digests, and publishes signed tree heads here:

1. Each accepted counter-changing event, and each identity event (an account enrollment, a per-epoch key issuance, a key registration), is serialized as a log leaf.
2. The API periodically builds a Merkle tree over the leaves and signs the root as a checkpoint with Ed25519.
3. This repository records those checkpoints in Git history and, for the checkpoint-covered entries, a manifest line per chunk naming its range, its length and its SHA-256; the chunk bodies are served from the host `entries/mirrors.json` names. Mature checkpoints are also anchored to Bitcoin through OpenTimestamps and to Sigstore Rekor.
4. The verifier refetches the leaves from the chunks the manifest names, recomputes every leaf hash, the hash chain linking them and the Merkle root, checks the signed checkpoint and the whole checkpoint archive, then folds the log back into counters. A shard admitted by digest is the shard this repository committed to, whichever host served it.

That means live counters are verifiable against the public log. A cached or served counter that does not match the fold of the signed log is detectable.

## What's here

Everything under `checkpoints/`, `ots/`, `entries/`, `revocations/`, `rekor/`, `swh/` and `jwks/` is written by the anchoring bot; `keys/` is published once by the operator when a key is created. What each file is for:

**`checkpoints/` — signed tree heads (STHs)**

- `latest.json` — the single newest checkpoint (`tree_size`, `root_hash`, `ts`, `signature`), pretty-printed. **Overwritten every checkpoint.** The O(1) entry point consumers and the verifier read first; also the git-published view that is compared against the live API to catch a split view.
- `<YYYY-MM-DD>.ndjson` — the permanent **append-only archive**, the browsable history of every STH ever signed: one compact JSON line per checkpoint, one shard per UTC day. Today's shard is appended to and freezes once the day rolls over.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

The newest checkpoint appears in both `latest.json` and the current shard on purpose: a moving pointer plus an append-only archive.

**`ots/` — Bitcoin timestamps (OpenTimestamps)**

A checkpoint's root is submitted to the OTS calendars, then matured into a proof once it lands in a Bitcoin block, so an anchored checkpoint `<tree_size>` produces:

- `<tree_size>.pending.json` — interim calendar receipt right after submission (queued, not yet in a Bitcoin block); superseded once the proof matures.
- `<tree_size>.ots` — the matured OpenTimestamps proof (standard binary `.ots`) anchoring that checkpoint's root in a Bitcoin block.
- `<tree_size>.json` — self-contained sidecar for that proof: the signed STH plus the Bitcoin block height, so a verifier needs nothing else to tie the `.ots` to a checkpoint. (The block height lives here, not in the binary `.ots`.)
- `latest.json` — pointer to the newest matured proof; overwritten as proofs mature.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

Not every checkpoint gets its own OTS proof — only the newest not-yet-submitted one each time submit runs; the rest ride a consistency proof to an anchored one.

**`entries/` — where the raw log leaves are, and what they must hash to**

The leaves are not stored in Git. A log of any size outgrows a repository, and the bodies are the bulk of it; what this repository keeps is the commitment to them, which is the part that has to be tamper-evident.

- `manifest/<first leaf>.ndjson` — one line per chunk: its first and last leaf, how many leaves, how many bytes, and the SHA-256 of the body. At most 1000 lines per file; the next file starts at the `to` of the last line of this one, so a reader walks the chain without listing a directory. Published once a checkpoint covers the leaves, so nothing named here is ever newer than the latest checkpoint: a just-cast reaction appears only after the next checkpoint seals it. Appends are batched, so the manifest also routinely trails that checkpoint by a few hundred leaves until the next batch lands, and an offline audit verifies the newest checkpoint the manifest fully covers and says which one.
- `mirrors.json` — the host the chunk bodies are served from. Anyone may keep a copy and publish their own file naming it; the digests above are what decides whether a copy is the real one, not the host it came from.
- Chunk bodies, at that host, as `<from>-<to>.ndjson` (zero-padded, e.g. `000000000001-000000000585.ndjson`), each holding the leaves one publish covered. Written once and never touched again. One JSON line per leaf. A chunk is immutable the moment it is written; the next publish starts a new one.
- Enrolment proofs, at the same host, as `proofs/<first two hex>/<sha256>.bin`. A proof is 14 KB and an ENROLL leaf names it by digest, so the leaf stays small and the body is fetched only by an audit that checks proofs.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

Because this repository commits to every leaf by digest, a clone of it plus the bodies from any host that serves them is a complete, independently archivable copy of the log, and the verifier can audit it **fully offline** (see Verify below). The API being unavailable, or serving something different, changes nothing about what this record proves - and neither does the mirror, since a body is admitted only if it hashes to what is committed here.

Each leaf is pseudonymous by design. For a signed reaction the `user_ref` field is the SHA-256 of a per-epoch client key: the extension mints a fresh key every epoch, the operator blind-signs it (so the log shows the key was issued to an enrolled account without showing which one), and the reaction carries the key's signature. For a reaction from an older client it is a rotating per-epoch pseudonym. Either way it is not your account, email, or any stable identifier: it changes every epoch and cannot be linked across epochs or back to a person, so mirroring the full log here exposes activity, never identities.

Revocations are part of the same log: account erasure and other public corrections are append-only `op=4` leaves, present in the chunks and listed in `revocations/latest.json`. The verifier checks that file against the actual `op=4` leaves covered by the signed root anchored here.

**`revocations/` — the tombstones, listed**

- `latest.json` — every `op=4` leaf the log holds, ascending by `seq`, with the tree it was written over. Nothing here is new: each field is already in the chunk line of the leaf it names. It exists so a reader who never folds the log can still see what was reversed and why, and so the verifier can check the list against the leaves.

**`keys/` — the operator's public identity keys**

- `blind-rsa-v1.json` — the RSA public key (SPKI) under which per-epoch client keys are blind-signed; the verifier pins it.
- `enroll-v1.vk` and `enroll-v1.json` — the verification key of the enrollment circuit (`enroll-v1`), its SHA-256, the `bb` version that produced it, the `salt_commitment` and the public-input layout. Rotating any of these is a genesis reset.

**`jwks/` — archived provider signing keys**

- `<provider>/<kid>.json` — the OpenID provider's public key that signed the token behind an enrollment leaf, archived the first time the key is seen so an old proof stays checkable after the provider rotates keys.

**`rekor/` — Sigstore Rekor anchors**

- `<tree_size>.json` — sidecar for a checkpoint anchored to [Sigstore Rekor](https://docs.sigstore.dev/logging/overview/), an independently operated public transparency log: `{tree_size, root_hash, rekor_uuid, rekor_log_index, rekor_url}`. The submitted entry carries the same signed tree head bytes, so Rekor independently witnesses each checkpoint's existence; the verifier cross-checks this by default (`--no-rekor` to skip), resolving the UUID and comparing the bytes.

**`swh/` — Software Heritage archival records**

- `latest.json` — the coordinate of the most recent daily Software Heritage save attempt for this repo: `{origin, git_commit, swhid, revision_url, save_request_id, save_request_status, save_task_status, snapshot_swhid, requested_ts}`. **Rewritten once per day** whether or not that day's save request was accepted (the `save_request_*` / `snapshot_swhid` fields are null when it was throttled, but `git_commit` / `swhid` always pin the day's head); its git history here is the append-only record of every daily save attempt.

Because this is a Git origin, the archived commit's SWHID is `swh:1:rev:<git_commit>` — SWHIDs are `sha1_git`, so the SWH revision hash *is* the git commit hash. That makes third-party preservation checkable rather than assumed: resolve `revision_url` (or the SWHID) against the Software Heritage API and confirm the archive holds this repo's history. The save runs asynchronously, so a just-published coordinate resolves once SWH completes the visit, the same "pending, then matures" shape as an OTS proof.

## Reading the commit history

Data appends are made by the anchoring bot; the docs, and the occasional operational reset, are committed by the operator. The emoji prefix on a bot commit says which stage of the pipeline it belongs to, so a glance down the history separates proofs still waiting on Bitcoin (⏳) from proofs already anchored (⚓):

| Prefix | Stage |
| --- | --- |
| 🌱 | raw leaves appended |
| 🔏 | signed artifact published |
| 🧭 | a pointer file moved |
| ⏳ | timestamp proof requested, not yet confirmed |
| ⚓ | timestamp proof confirmed |
| 📚 | third-party mirror refreshed |
| 🧹 | log wiped back to genesis |

The text after the prefix says what it did:

| Commit message | File written | What it means |
| --- | --- | --- |
| `🔏 add checkpoint 766` | `checkpoints/<YYYY-MM-DD>.ndjson` | a new Ed25519-signed tree head (STH) for `tree_size` 766 was appended — the substantive "a checkpoint was published" event |
| `🧭 update latest 766` | `checkpoints/latest.json` | the pointer to the newest checkpoint moved to 766 (the file the verifier reads) |
| `⏳ ots submit 759` | `ots/759.pending.json` | checkpoint 759's root was submitted to the OpenTimestamps calendars; awaiting a Bitcoin block |
| `⚓ ots anchor 759` | `ots/759.ots` | the proof matured — 759's root is now anchored in Bitcoin (the block height is recorded in the `ots/759.json` sidecar) |
| `⚓ ots sidecar 759` | `ots/759.json` | the self-contained sidecar for that proof (signed STH + block height) |
| `🧭 ots latest 759` | `ots/latest.json` | the pointer to the newest matured proof moved to 759 |
| `🌱 add entries 741-766` | `entries/manifest/<first leaf>.ndjson` | leaves 741–766 (now covered by a checkpoint) were published, and the manifest line now names the chunk holding them |
| `⛔ update revocations` | `revocations/latest.json` | a tombstone was added or the list was rebuilt over a newer tree — the file is rewritten whole, and only when its contents change |
| `🧭 update entries mirrors` | `entries/mirrors.json` | the host serving the chunk bodies was named, or changed |
| `⚓ rekor anchor 766` | `rekor/766.json` | checkpoint 766's signed tree head was submitted to Sigstore Rekor; the sidecar records the entry UUID |
| `📚 swh save a1b2c3d` | `swh/latest.json` | Software Heritage was asked to re-archive the repo; the record pins the archived commit `a1b2c3d` as `swh:1:rev:…` |
| `🔐 add jwks google/abc123` | `jwks/google/abc123.json` | a provider signing key was archived the first time an enrollment used it |
| `🔑 publish key blind-rsa-v1` | `keys/…` | the operator published a public key or verification key; an operator action, like a reset |
| `🧹 reset to genesis` | every generated file removed (`keys/` and `jwks/` survive) | the weekly wipe (and any on-demand one) — checkpoints, proofs and entries from before it are gone and `tree_size` restarts at 0 |

`tree_size` is the cumulative number of log leaves — it only ever grows between 🧹 resets, which on this staging log happen weekly (see **Staging notes** below).

**Commit messages are informational only.** The verifier never reads them: it recomputes everything from the file *contents* here plus the chunk bodies they commit to. Read them to follow what the bot did; nothing depends on their wording — including the prefixes, which older commits (published before they were introduced) don't carry.

**Why the numbers look out of order.** Checkpoint `tree_size` values jump by however many events landed in that hour (e.g. `742 → 754 → 759 → 766`), not by one. And an OTS submit always anchors the *newest* checkpoint not yet submitted (submits run right after each checkpoint), so several submits can walk newest to older (`766`, then `759`, …). Both are expected; most intermediate checkpoints never get their own OTS proof and are tied to an anchored one by consistency proofs instead.

**Editing this repository.** Docs (`README`, `LICENSE`, anything outside the data directories) are safe to edit: the verifier ignores them and the bot never touches them. The data directories (`checkpoints/`, `ots/`, `entries/`, `rekor/`, `swh/`) are machine-generated. Hand-editing them, force-pushing, or rewriting history is exactly the tampering the verifier is built to catch, and third-party mirrors preserve the real history. Don't edit them by hand.

## Verify

Set the coordinates of the deployment you are checking - every command below reuses them:

```bash
REPO=https://raw.githubusercontent.com/khasky/emojery-log-staging/main
KEY="--pubkey <staging-ed25519-key-base64>"   # published with the staging deployment
```

There is no API to point at. The verifier reads published files and nothing else: the checkpoint from `checkpoints/latest.json`, the leaves from the chunks `entries/manifest` names, each admitted only if its bytes hash to the digest committed here. Whether the service is up, down, or answering differently changes nothing about what a run proves.

### Full audit

Use the open-source verifier. It is run straight from its GitHub repository - there is no npm package, and a name that looks like one is not ours:

```bash
npx github:khasky/emojery-verifier --repo $REPO $KEY
```

That is the whole audit: every leaf rehashed, the hash chain replayed from genesis, the Merkle root recomputed and held against the signed checkpoint, the checkpoint archive replayed, the Rekor witness checked, the counters folded back out, and the identity invariants and enrolment proofs verified. It prints the recomputed totals as it goes - the numbers you can then publish without quoting us.

Add `--entries-base <url>` to read the bodies from your own copy instead of the host `entries/mirrors.json` names. The digests decide, so a mirror is checked exactly as strictly as the original.

A publish lands a tick behind the checkpoint that covers it, so the run audits the newest checkpoint the chunks fully cover and names both it and the tip. That is a smaller audit, not a failing one.

### Fast check

`--entries none` reads no leaves at all. It checks the signed checkpoints, their consistency proofs and the witnesses, which is seconds on a log of any size, and says plainly that the contents were not read:

```bash
npx github:khasky/emojery-verifier --repo $REPO $KEY --entries none
```

### Bitcoin anchor check

`--ots` additionally verifies the newest matured OpenTimestamps proof against a Bitcoin block. It is slower and can only pass after an OTS proof has matured, so it is separate from the audit above:

```bash
npx github:khasky/emojery-verifier --repo $REPO $KEY --ots
```

### From a checkout

The verifier lives in a separate public repository:

```
git clone https://github.com/khasky/emojery-verifier
cd emojery-verifier
pnpm install
node src/verify.mjs --repo $REPO $KEY
```

The production public key lives in one authoritative place: pinned in the [verifier source](https://github.com/khasky/emojery-verifier/blob/main/src/verify.mjs), and printed in that repository's README. It is deliberately not restated here, so a copy can't silently drift from the one the tool actually checks against. Any other deployment (staging, a fork, a different signing key) is verified by passing `--pubkey <base64>`, which is what `KEY` above carries. The identity pins beside it - the blind-signing key, the salt commitment, the admitted issuers and audiences - belong to that same deployment, so a run against any other needs `--blind-pubkey`, `--salt-commitment`, `--issuers` and `--audiences` too, and reports the checks it could not make as skipped rather than passed.

Expected successful output looks like this:

```
── Checkpoint ────────────────────────────────────────────────────────── 2 ✓
   ✓  checkpoint Ed25519 signature
   ✓  checkpoint is fresh (0.4h old, threshold 168h; a quiet log ages legitimately, tune --max-checkpoint-age-hours)

── Leaves & Merkle ───────────────────────────────────────────────────── 4 ✓
   ✓  every recomputed leaf_hash matches the served leaf (0 mismatch)
   ✓  fetched all 576 leaves (got 576)
   ✓  recomputed Merkle root == checkpoint root_hash
   ✓  hash chain replays from genesis (576 leaves, 0 break(s))

── Checkpoint archive ──────────────────────────────────────────── 7 ✓ · 1 ○
   ✓  checkpoint archive parses (1 STH line(s) in 1 shard(s))
   ✓  no two archived STHs disagree on one tree_size (0 conflict(s))
   ✓  every archived STH signature verifies (1 checked, 0 bad)
   ✓  archived STH timestamps are monotone in tree_size (0 regression(s))
   ✓  archive never exceeds the live tree (max archived 576 <= 576)
   ✓  the live checkpoint is present in the archive shards
   ✓  every archived root replays from today's leaves (1 checkpoint(s), 0 mismatch)
   ○  consistency chain (fewer than two archived checkpoints)

── Independent witness ───────────────────────────────────────────────── 4 ✓
   ✓  rekor sidecar 576 matches the archived checkpoint
   ✓  Rekor entry 108e9186e8c5... holds the STH bytes of checkpoint 576
   ✓  Rekor entry carries our Ed25519 checkpoint signature
   ✓  Rekor entry public key is the published log key

── Log semantics ───────────────────────────────────────────────── 3 ✓ · 1 ○
folded 255 (site,target,reaction) counters from 576 events
revocations: 0 tombstone(s)
   ✓  revocations/latest.json matches the op=4 leaves in the log (0)
   ✓  structural invariants hold (0 violation(s))
   ✓  account wipes are complete (0 violation(s); grace 48h)
   ○  served counts against the fold (no --counts-base)

── Identity ──────────────────────────────────────────────────────────── 5 ✓
identity: 74 enroll, 73 issue, 73 key leaf(s); 356 signed and 0 unsigned vote(s)
   ✓  identity invariants hold (0 violation(s))
   ✓  every signed vote verifies under its epoch key (356 checked, 0 bad)
   ✓  every ISSUE leaf is signed by its enrolled account key (73 checked, 0 bad)
   ✓  every KEY leaf carries a valid blind RSA-PSS signature (73 checked, 0 bad)
   ✓  every ENROLL proof verifies under the pinned verification key (74 ENROLL proof(s), 0 bad)

┌─ VERIFIED ───────────────────────────────────────────────────────────────┐
│  RESULT     PASS   25 passed · 2 skipped                                 │
│  tree size  576   root cb926d58c8...4f988f                               │
│  log key    79Wp/Myyu7... (--pubkey)                                     │
│  witnesses  Rekor 108e9186e8c5...                                        │
│  sources    khasky/emojery-log-staging@main · manifest + chunk bodies    │
│  elapsed    9.4s                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

A `○` is a check that was skipped and says why: a flag it needed was not passed, or the log does not carry what it would check. Exit code `0` means the verifier passed. Exit code `1` means the published entries, hash chain, signed checkpoint, checkpoint archive or revocation list did not match what the verifier recomputed. A mistyped flag is exit code `2` - no run silently checks less than you asked for.

## Reactions, removals, and tombstones

The log records counter-changing events, not just final state:

- `op=1` — a reaction was added.
- `op=2` — a reaction was changed; the leaf records both the new reaction and the previous one.
- `op=3` — a reaction was removed by the user.
- `op=4` — a revocation tombstone: a later public leaf that reverses an earlier `op=1`, `op=2`, or `op=3` leaf.
- `op=5` — an enrollment: a zero-knowledge proof that an OpenID provider (Google, Apple, Microsoft, Facebook, LinkedIn, Discord, Twitch or Slack) signed a token for this account, reduced to a `nullifier` that never reveals the provider's user id.
- `op=6` — a key issuance: the enrolled account identified by `nullifier` received one blind-signed per-epoch key (at most 3 per epoch).
- `op=7` — a key registration: a per-epoch public key with the operator's blind signature; every signed reaction points at one of these.

So a normal user "unreact" is `op=3`, not a tombstone. Tombstones are for append-only corrections such as account erasure or other public reversals. The original leaf stays in the log; the `op=4` leaf points at it with `revoke_seq`, and the verifier applies the inverse effect when recomputing counters.

## What this proves

The verifier checks integrity of the public counter history:

- the checkpoint signature matches the published Ed25519 key;
- the entries (from the chunks the manifest names) recompute to the signed Merkle root;
- their published hash chain replays from genesis, so their order is pinned as well as their contents;
- the root matches the checkpoint published in this repository;
- every checkpoint ever archived here replays from today's leaves — the whole published history lies on one append-only line;
- `revocations/latest.json` matches the actual `op=4` leaves in the log;
- the counters recomputed from the log are printed as the run folds them, so the totals can be republished by whoever ran the check;
- the checkpoint's Sigstore Rekor entry holds exactly its signed bytes (checked by default; `--no-rekor` to skip);
- with `--ots`, a matured checkpoint root is anchored in Bitcoin;
- every signed reaction carries a valid signature by a registered per-epoch key, every registered key carries the operator's valid blind signature, no epoch has more registered keys than issuances, every issuance cites an earlier enrollment by the same account key and carries that key's signature (at most `--keys-per-account`, 10 by default, per account per epoch), and every enrollment proof verifies against the pinned circuit key and the archived provider key (`--no-proofs` to skip the last one).

This does **not** prove that every reaction came from a unique human, or that the anti-abuse policy is perfect. It proves that the published counters match the public append-only log, that changes/removals/revocations are represented as verifiable log events, and that every signed reaction traces to an account opened with a real OpenID sign-in, so padding the counters would take real provider accounts and would be visible here.

## Administrators

This repository holds only the published log data and its documentation, no scripts or tooling. The backend automation performs operator maintenance (resetting the log to genesis, scaffolding a fresh empty layout), and it lands here as normal bot commits, visible in the commit history like everything else.

Force-pushing or rewriting history in this repo is itself the tamper signal, because third-party mirrors preserve the real history.

## Staging notes

Everything above describes this repository exactly as it describes the production log — the two READMEs are kept identical. What differs is only this: this repository is the log of the Emojery **staging** deployment at `https://api-staging.emojery.app`, not the production log at [emojery-log](https://github.com/khasky/emojery-log).

- **Own signing key.** Staging checkpoints are signed with a separate Ed25519 key, which the verifier does not pin — it pins the production key. Every verification run therefore needs `--pubkey`, which is what the `KEY` variable at the top of **Verify** carries. The production log needs no `--pubkey`.
- **Reset weekly.** This log, and the entire staging database behind it, is **fully reset to genesis weekly**, and on demand. So `tree_size` restarts from zero with each one.
- **Ephemeral by design.** Any checkpoint, proof, or entry here is wiped at the next reset and the log is rebuilt from scratch. Nothing in this repository is durable enough to cite or archive against — for that, use the production log.

## License

The log data in this repository is dedicated to the public domain under [CC0 1.0 Universal](LICENSE) — copy, mirror, and verify it freely.
