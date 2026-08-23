# Portal submission text

Ready-to-paste text for the GenLayer Portal submission form. The description below is exactly 996 characters (verified with Python `len()`, not eyeballed) against the Portal's 1000-character Notes/Description field limit.

## Name

```
UpgradeChangelogGate
```

## One-line summary

```
A reusable primitive that verifies a proposed upgrade's changelog faithfully matches its deterministic diff before a protocol's version pointer can move.
```

## Description / Notes field (996 characters)

```
UpgradeChangelogGate verifies a proposed upgrade's changelog is a faithful account of what actually changed before a protocol's version pointer can move. A protocol registers versioned JSON content; anyone may permissionlessly propose an upgrade backed by a GEN stake. The contract computes a deterministic field-by-field diff between old and new content in plain code -- never inside the non-deterministic block, since diffing already-agreed state needs no independent re-derivation. Validators independently judge whether the changelog honestly accounts for that diff, forced into FAITHFUL, INCOMPLETE, or MISLEADING; only that bucket is compared under the Equivalence Principle. MISLEADING forfeits the stake to the owner; otherwise it's refunded. Only FAITHFUL lets the owner apply the upgrade -- GenLayer decides only if the account is honest, never if the upgrade is wise. Uses run_nondet_unsafe with a hand-written leader/validator pair. 42 tests pass; genvm-lint and typecheck both clean.
```

## Source code / repository

```
https://github.com/Fortune9thx/upgrade-changelog-gate
```

## Deployed contract

**Network:** GenLayer Bradbury Testnet

**Contract address:** _to be filled in after deployment -- see CHANGELOG.md for the current live address once deployed._

## Category / tags

```
Governance, Upgrade Integrity, Equivalence Principle, Reusable Primitive, DAO Tooling
```

## Why this survives the current review bar

- **The diff cannot be gamed -- only the changelog about it can be dishonest.** The proposer supplies `new_content`, but the diff is computed against the protocol's actual stored content, not anything the proposer claims. There is no `expected_changelog` anywhere in storage; the only thing to compare a claim against is a fact the contract itself established.
- **Validators never trust the leader.** `validator_fn` calls the identical `leader_fn` again over the identical, already-agreed diff and changelog -- its own independent LLM judgment, not a shape check -- and only agrees if it lands on the same bucket.
- **Exactly one non-deterministic call per write method.** `evaluate_proposal` contains a single `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`; the diff itself is computed in plain deterministic code, since diffing already-agreed blockchain state has nothing that could legitimately vary between independent executions.
- **GenLayer's role is deliberately narrow.** The contract never asks whether an upgrade is wise -- only whether the changelog is honest about what changed. That is a bounded, checkable question with a ground truth (the diff) neither party authored alone.
- **Real consequential action, twice over.** Directly gates GEN stake routing (forfeiture vs. refund) and whether a protocol's actual content pointer can move at all.
- **Screened against a decision framework from an independently accepted Portal submission** (`spec-compliance-bounty`'s own `docs/DECISION_RECORD.md`), which itself independently identified and screened this exact underlying concept as passing every gate -- see [`docs/DESIGN.md`](docs/DESIGN.md#originality).

## Verification commands

```bash
genvm-lint check contracts/UpgradeChangelogGate.py
genvm-lint typecheck contracts/UpgradeChangelogGate.py
gltest tests/direct/ -v
```
