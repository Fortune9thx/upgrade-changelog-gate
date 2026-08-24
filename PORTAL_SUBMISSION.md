# Portal submission text

Ready-to-paste text for the GenLayer Portal submission form. The description below is exactly 988 characters (verified with Python `len()`, not eyeballed) against the Portal's 1000-character Notes/Description field limit.

## Name

```
UpgradeChangelogGate
```

## One-line summary

```
A reusable primitive that verifies a proposed upgrade's changelog faithfully matches its deterministic diff before a protocol's version pointer can move.
```

## Description / Notes field (988 characters)

```
UpgradeChangelogGate verifies a proposed upgrade's changelog is a faithful account of what actually changed before a protocol's version pointer can move. A protocol registers versioned JSON content; anyone may permissionlessly propose an upgrade backed by a GEN stake. The contract computes a deterministic field-by-field diff between old and new content in plain code -- never inside the non-deterministic block, since diffing already-agreed state needs no independent re-derivation. Validators independently judge whether the changelog honestly accounts for that diff: FAITHFUL, INCOMPLETE, or MISLEADING. MISLEADING forfeits the stake to the owner; otherwise it's refunded. Only FAITHFUL lets the owner apply the upgrade, bound to the exact protocol version the diff was computed against. Emits an event per write. Uses run_nondet_unsafe with a custom leader/validator pair. 50 tests pass; lint and typecheck clean. Live-verified: a real omitted change was correctly judged INCOMPLETE.
```

## Source code / repository

```
https://github.com/Fortune9thx/upgrade-changelog-gate
```

## Deployed contract

**Network:** GenLayer Bradbury Testnet

**Contract address:** [`0xfe4800F103D6BC5eC6E67938f10B63f178dcDb9e`](https://explorer-bradbury.genlayer.com/address/0xfe4800F103D6BC5eC6E67938f10B63f178dcDb9e)

Deploy tx `0x1bc422437e0e6c1d114bb26f36bf18eba906c139e9f145e20c0a9dad47d9168d` -- `ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`, confirmed readable (`get_proposal_count` returns `0`). This is the 1.2.0 redeployment, adding event emission on every write following a third benchmark pass against independently-accepted Portal repos -- see CHANGELOG.md. A full `register_protocol` → `propose_upgrade` → `evaluate_proposal` sequence was run live end to end against the 1.0.0 deployment (not just a bare deploy) -- real, unscripted model consensus correctly judged a silently-omitted admin-key change as `INCOMPLETE`, citing the exact undisclosed field. Both the 1.1.0 and 1.2.0 changes are purely deterministic (event emission included), so this record still stands as the live proof of the non-deterministic mechanism. Full record: `docs/DESIGN.md`'s "Live verification" section.

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
- **Strict security audit found and closed a real state-integrity bug, not just cosmetic ones.** `accept_proposal` originally had no protection against applying a stale proposal -- with permissionless proposing, a second FAITHFUL-verdict proposal computed against an earlier baseline could silently overwrite a more recent, already-applied upgrade. Fixed with a `based_on_version` check; regression-tested by actually reproducing the clobber scenario and proving it's now rejected. See [`docs/DESIGN.md`](docs/DESIGN.md#trust-boundaries).
- **Live-verified, not only mock-tested.** A full register → propose → evaluate sequence ran on real Bradbury consensus with a genuinely adversarial, unscripted scenario -- the model's real judgment even corrected an imprecise assumption in the original design narrative, disclosed openly rather than smoothed over. See [`docs/DESIGN.md`](docs/DESIGN.md#live-verification).
- **Benchmarked a third time, fresh, against the same accepted repos -- not just designed against them once.** Re-read `tendercouncil`/`spec-compliance-bounty`/`rubricproof-intelligent-contract` specifically for governance/staking/versioning patterns this contract's own shape would call for. Adopted event emission (matching `spec-compliance-bounty`'s convention) for lifecycle tracking; traced through and confirmed -- rather than assumed -- that a permissionless timeout-refund pattern from that same repo is unnecessary here, since `evaluate_proposal` already unconditionally resolves every staked GEN amount with no privileged party required. See [`docs/DESIGN.md`](docs/DESIGN.md#third-benchmark-pass-2026-08-23----re-screened-for-governancestaking-shaped-patterns).

## Verification commands

```bash
genvm-lint check contracts/UpgradeChangelogGate.py
genvm-lint typecheck contracts/UpgradeChangelogGate.py
gltest tests/direct/ -v
```
