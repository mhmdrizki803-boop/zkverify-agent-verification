# Verification-first agent design: what I check before an agent is allowed to say "done"

*Written for the zkVerify Agent Games — #AgentGames · @ZKVProtocol*

Most agent failures I have seen are not model failures. They are **verification failures**.

An agent that moves real funds on-chain has one job that matters more than reasoning quality: when it says "done", the world has to agree. Over months of running an autonomous agent 24/7 across multiple chains, the interesting engineering was never the prompt. It was building a system where the agent is structurally unable to lie to me by accident.

This is what that looked like.

## 1. Never trust a single source of state

The first real incident: an agent read a wallet balance from one public RPC, reported it, and acted on it. The number was wrong — the node was serving stale state. Nothing "failed"; the agent simply believed a source that had no second opinion.

The fix is boring and effective: every state read is cross-checked against at least three independent nodes. If they disagree, the read is rejected and retried. Agreement across independent operators is weak consensus, but it is dramatically better than a single point of trust, and it costs a few hundred milliseconds.

**Design rule:** a state read is not true because a node said so. It is true because independent sources agree.

## 2. "Submitted" is not "succeeded"

The second class of failure is subtler because the agent looks correct in the logs. A transaction is broadcast, the agent records "sent", and moves on. But broadcast is not inclusion, and inclusion is not success.

Every write now follows one fixed sequence:

1. simulate the call first, and refuse to send if the simulation reverts
2. check the nonce and the gas parameters against the current chain state
3. sign and send
4. **wait for the receipt**
5. require `status == 0x1` before anything downstream treats it as done

Step 4 and 5 are the ones people skip. Skipping them once means a retry after a timeout pays the fee twice for the same intent. The agent did exactly what it was told — it just had no concept of "confirmed".

**Design rule:** the agent's unit of completion is a receipt, not a request.

## 3. Verify signatures yourself, even when the server says "ok"

Authentication flows hand you a challenge, you sign, and a server answers "verified". That answer is a claim, not a fact, and it is produced by the party that benefits from you believing it.

So we recover the signer locally: rebuild the typed data from the challenge, recover the address from the signature, and compare it to the wallet we expect. If our own check disagrees with the service, the service is wrong — and we have caught exactly that class of mismatch, including cases where the identity being verified was not the one the client thought it signed for.

**Design rule:** verification you cannot reproduce locally is not verification, it is trust.

## 4. Secrets that touch a transcript are already compromised

An agent's context window is a public surface in practice: prompts are logged, logs are shipped, transcripts are pasted into bug reports. Any key that enters that surface should be treated as burned.

Keys live in files with `0600` permissions and are read at use time. The agent itself only ever handles a short fingerprint (a prefix and a suffix) for display and comparison. The rule is absolute because the failure is silent: a leaked key does not break anything, it just waits.

**Design rule:** the agent never holds the secret, it holds a reference to it.

## 5. An agent that grades its own work is having a monologue

The deepest failure mode is structural: the agent decides what success looks like, performs the work, and then evaluates itself. Every failure becomes a rounding error in its own narrative.

Verification has to come from outside the agent. In practice that means at least one artifact the agent cannot author: a transaction receipt from the chain, a state diff produced by someone else's node, a signature we recovered independently, a balance change read from a second provider.

## Where zkVerify fits

Everything above works because the checks are cheap relative to the work: a receipt is a few hundred bytes, a signature recovery is milliseconds. What it does not solve is the case where the check itself is expensive or private — where confirming that an agent did the right thing would require re-running its computation, or revealing the data it computed over.

That is exactly the gap zero-knowledge proofs close, and it is why the framing behind zkVerify is the right one for agent infrastructure. A proof turns "trust me, I ran the logic" into "check this, it is small, and it reveals nothing about how I decided". The verifier does not need to re-execute the agent, does not need its logs, and does not need its secrets. It needs a proof and a verifier.

For agent competitions this is not a nice-to-have. If agents are scored on outcomes, the scoring layer must be able to verify what happened without trusting the agent's own report. Otherwise the leaderboard is a self-assessment contest.

## The rule that survived everything

After all the incidents, one principle absorbed most of the others:

**Agents should emit evidence, not assertions.**

"I did X" is worth nothing. "Here is the transaction hash, here is the state diff, here is the independent check that confirms it" is auditable — by me, by a judge, by a stranger with a block explorer.

That is the direction agent infrastructure should move in: not agents that act faster, but agents whose work can be checked by someone who does not trust them. The Agent Games are a good bellwether for it, because they put verification at the centre of the scoring instead of the edge.

---

*This entry was written for the zkVerify Agent Games Creator Track. #AgentGames*

*Thread version on X: [link to thread]*
