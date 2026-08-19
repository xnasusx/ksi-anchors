# ksi-anchors

Append-only anchor logs for [`ksi-harness`](https://github.com/RootCawsLLC/ksi-harness) evidence
lockers.

## What is in here

One JSON Lines file per monitored boundary, under `anchors/`. Each line records what a single
collection produced:

```json
{"schema":"ksi-harness/evidence-anchor/1","anchored_at":"…","root_sha256":"…",
 "bundle_count":42,"checks":{"aws.iam.mfa-coverage":3,"…":1},"previous_sha256":"…"}
```

The manifest root, the total bundle count, and the **per-check run counts** — plus the hash of the
entry before it, so the log cannot be silently trimmed either.

## Why it is a separate repository

Everything protecting an evidence locker — the per-check hash chain, the signed manifest, the
RFC 3161 token — lives *inside* the locker. So it is protected against alteration and not against
removal: delete two-thirds of a check's history, regenerate the manifest, and every integrity check
still passes, because a chain proves the consistency of what remains and says nothing about what is
gone.

This log is what notices. It is small on purpose: it moves the thing that has to survive from
megabytes of bundles to one line per collection, which is small enough to keep somewhere the
evidence writer cannot reach.

Held beside the evidence, it is removed by whatever removes the evidence, and the whole exercise
reduces to storing a hash next to its own data. That is why this is a different repository under a
different account rather than a file in the harness.

## What this arrangement does and does not give you

**Does:** a principal who can write an evidence branch cannot rewrite the record of how much
evidence there was. A restored, mirrored, or exported copy of a locker is reconciled against a
record that did not travel with it. Deleting the evidence does not delete the account of it.

**Does not:** true independence. The workflow that writes the evidence also holds the token that
writes here, so anyone able to modify that workflow can write both sides. This is separation of
storage and credential, not separation of control. Genuine independence needs an appender the
evidence writer cannot influence at all — an append-only endpoint that rejects rewrites regardless
of caller, or an assessor's own copy.

Reasoning in full: [ADR 0007](https://github.com/RootCawsLLC/ksi-harness/blob/main/docs/adr/0007-anchor-log.md).

## Rules for this repository

- **Nothing here is ever edited or deleted.** A line that is wrong stays, and a correcting line is
  appended after it. The log is chained; rewriting one entry invalidates every entry after it, which
  is the property that makes it worth keeping.
- **Nothing else lives here.** No evidence, no reports, no scripts. A repository that acquires other
  purposes acquires other people with write access.
- **Force-pushing to `main` should be disabled**, and history should never be rewritten. An anchor
  whose history can be replaced is not an anchor.
- **`*.jsonl` is marked `-text`** in `.gitattributes` so no client translates line endings. These are
  chained records; a file whose bytes depend on which platform last touched it is a poor thing to be
  reasoning about deletion with.

## Reading it

```bash
# every anchored root for a boundary, oldest first
jq -r '"\(.anchored_at)  \(.root_sha256[0:12])  \(.bundle_count) bundle(s)"' anchors/<boundary>.jsonl

# reconcile a locker against it
ksi verify --evidence .evidence --anchor anchors/<boundary>.jsonl
```

`ksi verify --anchor` reports three distinct states: `root_unknown` (a manifest root never recorded
here), `shrunk` (fewer runs than were anchored), and `missing_check` (a check this log knows and the
locker no longer contains). Growth is never a finding — a locker is supposed to grow.
