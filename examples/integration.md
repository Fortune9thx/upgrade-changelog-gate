# Integration example

UpgradeChangelogGate is a reusable primitive with no domain-specific fields -- `protocol_id`, `content`, and `changelog` are entirely caller-supplied, so any upgradeable protocol can adopt it as its changelog-honesty gate without modification.

## Important: writes must be called directly, not cross-contract

`propose_upgrade`, `evaluate_proposal`, and `accept_proposal` are all `@gl.public.write` methods. GenLayer Bradbury's cross-contract *write* calls (`.emit(...)`) are known to silently no-op -- the caller's own transaction reaches `ACCEPTED` with no error, but the target contract's state never actually changes. Cross-contract *reads* via `.view()` are reliable. This means:

- A DAO's governance flow should call `propose_upgrade`/`evaluate_proposal`/`accept_proposal` as real, directly-signed transactions -- from a delegate's wallet, a keeper script, or a multisig's own transaction, never triggered inline from inside another Intelligent Contract's write execution.
- A protocol that wants to *react to* an applied upgrade (e.g. actually pull the new fee parameter into its own state) should be a **pull-based** consumer: it reads this contract's finalized `get_protocol` state via `.view()` when someone explicitly asks it to, rather than expecting to learn about the change automatically or inline.

## Example: a DAO's upgrade flow via genlayer-js

```javascript
import { createClient } from "genlayer-js";
import { testnetBradbury } from "genlayer-js/chains";

const client = createClient({ chain: testnetBradbury, account, provider });

// One-time: register the protocol with its current parameters.
await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "register_protocol",
  args: [
    "my-protocol",
    JSON.stringify({ fee_bps: "30", admin: "0xCurrentAdmin..." }),
    100_000000000000000000n, // min_proposal_stake, in wei
  ],
});

// A delegate proposes an upgrade, staking the required minimum.
const proposalId = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "propose_upgrade",
  args: [
    "my-protocol",
    JSON.stringify({ fee_bps: "50", admin: "0xCurrentAdmin..." }),
    "Increased fee_bps from 30 to 50 to fund development.",
  ],
  value: 100_000000000000000000n,
});

// Anyone triggers evaluation (the one consensus round).
const verdict = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "evaluate_proposal",
  args: [proposalId],
});
// verdict === "FAITHFUL" | "INCOMPLETE" | "MISLEADING"

// Only the protocol owner, and only if verdict === "FAITHFUL":
if (verdict === "FAITHFUL") {
  await client.writeContract({
    address: GATE_ADDRESS,
    functionName: "accept_proposal",
    args: [proposalId],
  });
}
```

Note: pass `new_content`/`initial_content` as JSON-encoded strings (via `JSON.stringify`), never as native JS objects -- the contract parameter type is `str`, which it parses and validates itself. The `genlayer` CLI's `write --args` flag re-parses any JSON-shaped argument string automatically, which is convenient for these string-typed JSON parameters (unlike a genuinely array-typed parameter, which the CLI handles differently) -- either the CLI or `genlayer-js` work fine here.

## Example: a DAO contract pulling the applied result

```python
# Inside a separate DAO contract's own write method, in its deterministic
# section (never inside a non-deterministic block -- cross-contract calls
# are forbidden inside run_nondet_unsafe closures):
import json
import genlayer.gl as gl
from genlayer import Address

GATE_ADDRESS = Address("<deployed UpgradeChangelogGate address>")

def sync_fee_parameter(self, protocol_id: str):
    raw = gl.get_contract_at(GATE_ADDRESS).view().get_protocol(protocol_id=protocol_id)
    protocol = json.loads(raw)
    self.current_fee_bps = protocol["content"]["fee_bps"]
    self.synced_version = protocol["version"]
```

This pull pattern -- UpgradeChangelogGate records the verified, applied content; a consumer contract reads it via `.view()` on demand -- is the verified-reliable shape for composing this primitive into a larger system on the current GenLayer Bradbury build.

## If evaluation never resolves

`evaluate_proposal` can, in principle, fail to reach validator agreement (a genuinely ambiguous case, or a transient infrastructure issue) -- when that happens, no state is written and the proposal stays `"pending"`, simply retriable by calling `evaluate_proposal` again. If it's been `"pending"` for at least 72 hours with no agreed verdict, anyone -- including the original proposer -- may call `reclaim_expired_proposal(proposalId)` to refund the staked GEN and mark the proposal `"expired"`. A consuming DAO's keeper/automation should treat a proposal that's been pending unusually long as a signal to retry `evaluate_proposal` first, and only fall back to `reclaim_expired_proposal` once the 72h window has genuinely elapsed.
