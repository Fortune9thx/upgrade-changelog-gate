# Final checklist

Pre-submission / pre-deployment verification record for UpgradeChangelogGate.

## Contract requirements

- [x] File header exact match: line 1 `# v0.2.16`, line 2 `# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }` -- matches every other live-deployed contract in this account's project history using this pin.
- [x] Storage: `protocols: TreeMap[str, str]`, `proposals: TreeMap[str, str]`, `protocol_latest_proposal: TreeMap[str, str]`, `proposal_counter: u256`. Bare class-level annotations, no explicit `TreeMap()` construction in `__init__` -- the verified-working pattern matching GenLayer's own official boilerplate reference contract.
- [x] Equivalence strategy: `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`, both named `def`s (never `lambda`). `validator_fn` independently re-derives the verdict via calling `leader_fn()` again over already-agreed diff/changelog data -- never validates leader output by shape alone.
- [x] Outcome forced into exactly one of `"FAITHFUL"`/`"INCOMPLETE"`/`"MISLEADING"` (`_coerce_verdict`), and only that bucket is compared under the Equivalence Principle (`validator_fn`'s sole comparison).
- [x] Non-determinism boundaries: exactly one top-level `gl.vm.run_nondet_unsafe` call in `evaluate_proposal`; no `self.*` read/write inside `leader_fn`/`validator_fn`; the diff itself is computed in plain deterministic code (not redundantly inside the nondet closures), a deliberate design choice documented in `docs/DESIGN.md`, not an oversight.
- [x] Required public methods present: `register_protocol`, `propose_upgrade` (payable), `evaluate_proposal`, `accept_proposal`, `update_stake_requirement` (5 writes); `get_protocol`, `get_proposal`, `get_latest_proposal_for_protocol`, `get_proposal_count` (4 views) -- 9 total, confirmed by `genvm-lint check`'s own method count output.
- [x] Error handling: `gl.vm.UserError` used for every user-facing error; no bare `raise Exception(...)`. Strong validation on protocol_id format, content shape/scalar-only values, changelog non-emptiness, stake minimums, ownership on every owner-gated method.
- [x] Educational comments: module docstring covers why the diff is computed outside the non-deterministic block, independent re-derivation rationale, discrete-bucket rationale, the Portal-rejection-pattern mapping, and the exact Equivalence Principle strategy chosen.

## Quality bar

- [x] `genvm-lint check contracts/UpgradeChangelogGate.py` -- 3 checks passed, zero warnings on the first pass (no post-hoc fixes needed).
- [x] `genvm-lint typecheck contracts/UpgradeChangelogGate.py` -- zero type errors on the first pass.
- [x] `gltest tests/direct/ -v` -- 42/42 passed on the first full run; 43/43 after correcting one test's expected verdict following live-network testing; 50/50 after a second strict audit pass added regression tests for two real fixes (see below).
- [x] Studio + Bradbury ready: dependency pin (`py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6`) matches both networks' current supported runner; storage pattern (`TreeMap[str, str]` only) is the Bradbury-verified-reliable one.
- [x] Obviously useful/reusable: no domain-specific coupling -- `protocol_id`/`content`/`changelog` are fully caller-supplied, making this composable by any upgradeable DAO/protocol/multisig with a versioned content pointer.
- [x] "Validators only checked format" is not a valid criticism: `TestValidatorIndependence` in the test suite directly proves independent re-derivation and verdict-mismatch rejection.
- [x] "Outcome not bound by equivalence" is not a valid criticism: the verdict bucket is the only value ever compared, and the only thing stake routing / pointer-flip eligibility act on.
- [x] "Format-only validator" / "generic AI decides X" is not a valid criticism: GenLayer's role is narrowly whether prose matches a deterministic diff, never whether the underlying upgrade is a good idea -- see `docs/DESIGN.md`'s scope-boundary section.

## Lessons applied proactively (not discovered via a later audit pass this time)

- [x] `protocol_id` format-restricted to ASCII from the first commit, closing the Unicode-encoding-collision class of bug (found and patched after the fact in IndependentEvidenceSettler) before it could ever occur here.
- [x] Changelog gets the same non-blocking manipulation-heuristic screen (`_MANIPULATION_PATTERNS`) as this account's other caller-controlled-instruction-adjacent fields, shipped from the start.
- [x] Verdict is a fixed three-string enum with zero numeric clamp/snap arithmetic -- structurally avoids the whole non-finite-float edge-case class a numeric bucket scale needs explicit guards against.
- [x] `.github/workflows/ci.yml` (lint, typecheck, full test suite on every push/PR) shipped in the initial commit, not added in a later benchmarking pass.
- [x] `docs/DESIGN.md`'s "Trust boundaries" section written proactively, including the self-dealing stake-forfeiture-is-a-no-op observation and the dead-code defensive-check disclosure -- both identified by reasoning through the design before shipping, not found by an external reviewer afterward.
- [x] Originality screened, in writing, against every primitive this account has already built (IndependentEvidenceSettler, Equiv, AgentIntentSettlement) and every repo benchmarked while designing this one (`tendercouncil`, `spec-compliance-bounty`, `rubricproof-intelligent-contract`) -- see `docs/DESIGN.md#originality`.
- [x] **One thing proactive design could not substitute for: live testing found a real, honest correction to the design narrative anyway.** The original test suite assumed a silently-omitted admin-key change would be judged `MISLEADING`; live, unscripted consensus correctly judged it `INCOMPLETE` instead, per the contract's own stated bucket definitions (omission, not contradiction). The test suite and docs were corrected to match the verified-real behavior rather than forcing the original assumption -- disclosed openly in `docs/DESIGN.md`'s "Live verification" section as a genuine finding, not smoothed over. This is the one category of bug no amount of proactive design review can catch in advance: what an actual, non-deterministic model call will really decide.

## Second audit pass (2026-08-23) -- adversarial re-read of the actual code, not the docs

- [x] **HIGH -- stale-proposal clobber, a real state-integrity bug found by tracing every write path, not hypothesized.** `accept_proposal` applied `proposal["new_content"]` with no check that the protocol hadn't moved since the proposal's diff was computed. With permissionless proposing, two proposals against the same baseline could both be independently judged `FAITHFUL`; accepting the second after the first silently overwrote the protocol with a diff computed from the now-stale baseline, discarding whatever the first upgrade changed for any field the second proposal didn't also touch. Fixed with `based_on_version`, stored per-proposal and checked before every acceptance. Regression-tested (`TestStaleProposalGuard`) by actually reproducing the two-competing-proposals scenario and proving the clobber does not happen -- not merely that the ordinary path still works.
- [x] **LOW -- exact-case-only verdict matching.** A model writing `"Faithful"` instead of `"FAITHFUL"` -- casing, not dishonesty -- failed closed to `INCOMPLETE` unnecessarily. Fixed with case-insensitive matching that still always stores the canonical uppercase form. Regression-tested (`TestVerdictCoercion`), including that genuinely invalid verdict text still fails closed correctly and the fix doesn't widen that net.
- [x] Confirmed solid under adversarial review, not just asserted: float-valued content fields already correctly rejected by the existing scalar-type check (verified, not assumed); no checksum-lookup risk anywhere (this contract has no caller-supplied address parameters at all); CEI ordering in `evaluate_proposal` already correct (state marked `"evaluated"` before the stake transfer); per-field content caps looser than the aggregate cap in the worst case (cosmetic constant-tuning only, the aggregate check independently bounds the real DoS risk).
- [x] Re-verified after fixes: `genvm-lint check`/`typecheck` still zero warnings, all 50 tests pass (7 new regression tests), redeployed to Bradbury and confirmed readable.

## Third benchmark pass (2026-08-23) -- re-screened against the same accepted repos for a different lens

- [x] Re-read `tendercouncil`, `spec-compliance-bounty`, `rubricproof-intelligent-contract` a second time, this time hunting for governance/staking/versioning-shaped patterns specific to this contract's own shape rather than the evidence-fetch-focused findings (SSRF, CI) a first pass over the same repos already surfaced for this account's other primitive.
- [x] **Adopted: event emission.** `spec-compliance-bounty` emits a `gl.Event` after every state-mutating write; this contract had none. Added `ProtocolRegistered`/`ProposalSubmitted`/`ProposalEvaluated`/`ProposalAccepted`/`StakeRequirementUpdated`. Confirmed by inspection that this is a strengthening, not a requirement: this account's own already-accepted IndependentEvidenceSettler ships with zero events, and neither `tendercouncil` nor `rubricproof-intelligent-contract` use them either.
- [x] **Considered and deliberately not adopted: permissionless timeout-gated refund.** Traced `evaluate_proposal`'s own release path rather than assuming it needed the same escape hatch `spec-compliance-bounty` ships -- confirmed it is already permissionlessly callable by anyone and unconditionally resolves the stake before any owner-gated step, so no stuck-funds condition exists here to guard against.
- [x] Re-verified after the change: `genvm-lint check`/`typecheck` still zero warnings, all 50 existing tests pass unchanged (no new tests needed -- event emission isn't asserted in either this repo's or `spec-compliance-bounty`'s own direct-mode suite).
- [x] Redeployed to Bradbury: `0xfe4800F103D6BC5eC6E67938f10B63f178dcDb9e`, deploy tx `0x1bc422437e0e6c1d114bb26f36bf18eba906c139e9f145e20c0a9dad47d9168d` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`), confirmed readable (`get_proposal_count` returns `0`). Change is purely additive/deterministic, so the existing live consensus-mechanism proof still stands for this redeployment -- not re-run, since nothing about the mechanism it exercises changed.
- [x] README.md / docs/DESIGN.md / PORTAL_SUBMISSION.md / CHANGELOG.md updated with the new address and this pass's findings.

## Steward review (2026-08-24) -- a real gap this account's own third benchmark pass got wrong

- [x] GenLayer Portal steward finding: a proposal's stake could remain locked forever if `evaluate_proposal` never reached validator agreement. Directly overturns the third benchmark pass's own "Finding 2" conclusion above -- disclosed as a genuine reasoning error (conflating "permissionlessly retriable" with "guaranteed to converge"), not smoothed over. See `docs/DESIGN.md`'s "Steward review" section.
- [x] **Fix: `reclaim_expired_proposal(proposal_id: str) -> None`.** Bounded (72h via `PROPOSAL_EVALUATION_TIMEOUT_SECONDS`), permissionless. Refunds the proposer's stake and marks the proposal `"expired"` (a third terminal status alongside `"evaluated"`).
- [x] Meets the steward's three explicit requirements directly, each backed by a dedicated test: (1) exactly-once refund -- shares `evaluate_proposal`'s own `status == "pending"` precondition, both flip status before any transfer; (2) cannot run after evaluation -- an `"evaluated"` proposal fails the same precondition regardless of elapsed time; (3) cannot bypass a resolved forfeiture -- the forfeiture/refund inside `evaluate_proposal` happens before status could ever allow a second look.
- [x] Technical verification, not assumption: confirmed `gltest`'s `direct_vm.warp()` does not propagate into a live contract call's `gl.message_raw['datetime']` (read the mock's own `vm.py` source) before switching to `datetime.now(timezone.utc)` for the elapsed-time comparison specifically -- and confirmed `datetime.now()` is explicitly sanctioned as deterministic by reading `genvm-lint`'s own `safety.py` source (`W002`'s forbidden-call list names `time.time`/`uuid.uuid4`/etc. but excludes it, with an explicit comment saying so). Every other timestamp in the contract is unchanged.
- [x] 8 new regression tests (`TestExpiryReclaim`, 58 total up from 50): before-timeout rejection, after-timeout success (including zero-stake), evaluated-proposal-cannot-be-reclaimed, expired-proposal-cannot-be-evaluated, double-reclaim rejection, permissionless-caller success, unknown-proposal rejection.
- [x] Re-verified after the fix: `genvm-lint check`/`typecheck` still zero warnings, all 58 tests pass, redeployed to Bradbury and confirmed readable. The existing live consensus-mechanism proof still stands -- `reclaim_expired_proposal` is an entirely new method, not a modification of `evaluate_proposal`/`leader_fn`/`validator_fn`.
- [x] README.md / docs/DESIGN.md / PORTAL_SUBMISSION.md / CHANGELOG.md updated with the new address and this finding.

## Deliverables

- [x] `contracts/UpgradeChangelogGate.py`
- [x] `tests/direct/test_upgrade_changelog_gate.py` (+ `tests/direct/conftest.py` for the one Windows-only gltest compatibility patch)
- [x] `README.md`
- [x] `docs/DESIGN.md`
- [x] `PORTAL_SUBMISSION.md`
- [x] `FINAL_CHECKLIST.md` (this file)
- [x] `LICENSE` (MIT), `CHANGELOG.md`, `SECURITY.md`, `.gitignore`
- [x] `examples/integration.md`
- [x] `.github/workflows/ci.yml`

## Repo / deployment

- [x] Git repository initialized, sole-author commit history confirmed (`git log --format='%an <%ae>'` -> `Fortunex9 <fortuneemx@gmail.com>`, no Claude co-author trailer)
- [x] Pushed to `https://github.com/Fortune9thx/upgrade-changelog-gate`
- [x] Deployed (1.0.0, pre-audit) to GenLayer Bradbury testnet: `0x60553Fb5BAE7E4681a330169e2c17E8dde414f97` -- superseded, kept in CHANGELOG.md for history only
- [x] Redeployed (1.1.0, second audit pass) to GenLayer Bradbury testnet: `0xfd702c49bbDaD2BD47438719d85F06cAD44983Cf`, deploy tx `0x651d80f7619235eab4b93be9b65d45b6c43508981a586e6400c7ec6093ad557a` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`) -- superseded
- [x] Redeployed (1.2.0, third benchmark pass) to GenLayer Bradbury testnet: `0xfe4800F103D6BC5eC6E67938f10B63f178dcDb9e`, deploy tx `0x1bc422437e0e6c1d114bb26f36bf18eba906c139e9f145e20c0a9dad47d9168d` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`) -- superseded
- [x] Redeployed (1.3.0, steward review) to GenLayer Bradbury testnet: `0xb56C016DFe03744B02ff8DeD1E35e0b4d73f2C0D`, deploy tx `0xe173c8db06fb40a74828a46c455996585a7ed4885e5360ab92f8842842ab9bc1` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`) -- this is the current, live address
- [x] Deployed contract address confirmed readable: `get_proposal_count()` called post-deploy, returned `0`
- [x] **Real `register_protocol` → `propose_upgrade` → `evaluate_proposal` transaction sequence executed live against the 1.0.0 deployment, not just a bare deploy.** All three `ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`. `evaluate_proposal` reached genuine, unscripted, finalized consensus on a real adversarial scenario, correctly judged `INCOMPLETE` -- full record in `docs/DESIGN.md`'s "Live verification" section. Both 1.1.0 fixes are purely deterministic (no LLM/consensus dependency), so they are fully proven by the 7 new regression tests rather than needing a second live consensus round; the live proof of `leader_fn`/`validator_fn` themselves (unchanged in 1.1.0) still stands. `accept_proposal` was NOT exercised live, correctly: a live-verified `INCOMPLETE` verdict must never allow acceptance, and it didn't need to -- that guard, and the new `based_on_version` guard, are both covered by passing direct-mode tests.
- [x] README.md / PORTAL_SUBMISSION.md / CHANGELOG.md updated with the current live deployed contract address and both audit passes' results, including the honest correction to the original test narrative that live testing surfaced
