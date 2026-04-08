# Invalidation Library — Generic Reasons for Web3 Security Issue Rejection/Downgrade

> **Usage**: The Step 3 Selector Agent reads this file and picks the 5 most applicable reasons for the issue under review. Each reason is self-contained — the agent matches based on the issue's characteristics.

---

## UNREALISTIC_PRECONDITIONS

**UP-1: Requires extreme token decimals**
The attack only works with tokens that have unusual decimal configurations (e.g., >24 or 0 decimals). Most real-world tokens use 6 or 18 decimals. If the attack requires non-standard decimals to produce meaningful impact, the likelihood is negligible in practice.

**UP-2: Requires attacker to hold majority of supply**
The attack assumes the attacker controls >50% of a token's circulating supply. Acquiring this position would be prohibitively expensive for any token with meaningful market cap and would itself move the price against the attacker.

**UP-3: Requires specific block.timestamp or block.number**
The attack depends on a precise block timestamp or number being reached. Validators/miners have limited control over timestamps (~15s drift on Ethereum), and exact block number targeting is impractical without validator collusion.

**UP-4: Requires unrealistic initial deposit or position size**
The described attack needs the attacker to be the first depositor or to create a position so large that it would require more capital than exists in the protocol or related markets.

**UP-5: Requires multiple low-probability events to coincide**
The attack path requires several independent unlikely conditions to all be true simultaneously. The combined probability makes the attack practically infeasible even if each condition alone is possible.

---

## COST_EXCEEDS_PROFIT

**CP-1: Gas cost exceeds extracted value**
The attack requires many transactions or complex operations whose cumulative gas cost exceeds the value that can be extracted. Calculate: total gas used × gas price vs. profit. If gas ≥ profit under normal gas prices, the attack is economically irrational.

**CP-2: Flash loan fees make attack unprofitable**
The attack relies on flash loans but the flash loan fees (typically 0.05-0.09%) on the required borrow amount exceed the profit from the exploit. The report ignores these fees in its profit calculation.

**CP-3: Slippage on required swaps exceeds profit**
The attack requires large swaps that would incur significant slippage on real DEX liquidity. The report assumes zero slippage or uses infinite liquidity assumptions that don't hold in practice.

**CP-4: Capital lockup cost exceeds grief value**
The attack requires the attacker to lock significant capital for an extended period. The opportunity cost of that capital (vs. staking, lending, or simply holding) exceeds the damage inflicted on the victim.

**CP-5: Attack requires sustained spending over multiple blocks**
The exploit requires the attacker to spend resources across many blocks or transactions over time. The cumulative cost grows while the extractable value remains fixed or diminishes, making the attack a net loss.

---

## DESIGN_TRADEOFF

**DT-1: Behavior is intentional gas optimization**
The "vulnerable" pattern exists to save gas. The alternative (which would prevent the described issue) costs significantly more gas per transaction. The protocol chose the cheaper path knowing the theoretical risk, as the practical impact is negligible.

**DT-2: No alternative solution with better security properties**
The described behavior is inherent to the mathematical/economic model used. Every known alternative model has the same or worse tradeoff. The report identifies a theoretical limitation, not a fixable bug.

**DT-3: Known limitation documented in protocol design**
The protocol's documentation, comments, or design documents explicitly acknowledge this behavior as a known limitation. The team made a conscious decision to accept this tradeoff.

**DT-4: Fixing would break composability or core functionality**
The suggested fix would break the protocol's composability with other DeFi protocols, or would require fundamentally changing core functionality that other features depend on.

---

## EXISTING_GUARD

**EG-1: Access control prevents unauthorized caller**
The vulnerable function has an access modifier (onlyOwner, onlyRole, etc.) or equivalent check that prevents the attacker described in the report from calling it. The report assumes the attacker can call a restricted function.

**EG-2: Reentrancy guard blocks the described path**
The contract uses a reentrancy guard (ReentrancyGuard, nonReentrant, lock modifier, or checks-effects-interactions pattern) that prevents the described reentrant call sequence.

**EG-3: Minimum amount or threshold check blocks dust attacks**
The function enforces a minimum deposit, withdrawal, or transfer amount that prevents the dust-amount manipulation described in the report.

**EG-4: Pause mechanism provides emergency mitigation**
The protocol has a pause/emergency mechanism that governance can trigger to halt the vulnerable operation before significant damage occurs. While not a fix, it bounds the maximum exploitable window.

**EG-5: Validation check on input parameters prevents the attack vector**
The function validates its inputs (e.g., non-zero address, bounded range, valid enum) in a way that blocks the specific input the attack requires. The report misses this validation.

---

## UNREACHABLE_STATE

**US-1: Required state combination is prevented by invariant**
The attack requires two state variables to have specific values simultaneously, but the protocol maintains an invariant that makes this combination impossible. Other functions that modify these variables always maintain the invariant.

**US-2: Previous operation always resets the vulnerable variable**
The attack requires a specific state left over from a previous operation, but that operation always resets or clears the variable before returning. The "stale state" the attack depends on is never actually stale.

**US-3: Initialization prevents the zero-state attack**
The attack exploits an uninitialized or zero state, but the contract's constructor, initializer, or factory always sets the state to a safe non-zero value before any user interaction is possible.

**US-4: Sequence of operations required is blocked by intermediate checks**
The attack describes a multi-step sequence, but an intermediate step has a check that fails given the state produced by the previous step. The full sequence cannot complete.

---

## SELF_HARM_ONLY

**SH-1: Attacker can only reduce their own balance**
The described manipulation only affects the attacker's own position or balance. No other user's funds are at risk. Self-inflicted losses are not a valid security finding.

**SH-2: Grief requires attacker to lock own funds permanently**
To execute the grief, the attacker must permanently sacrifice or lock their own funds in an amount equal to or greater than the damage caused. This is economically irrational unless the attacker's goal is purely destructive with no financial motive.

**SH-3: Attack outcome is equivalent to a donation**
The net effect of the attack is that the attacker transfers value to other users or the protocol without receiving anything in return. This is functionally a donation, not an exploit.

---

## DUST_IMPACT

**DI-1: Rounding error bounded to 1 wei per operation**
The described rounding error produces a maximum discrepancy of 1 wei (or smallest unit) per operation. This does not compound meaningfully even over millions of operations due to the bounded per-operation error.

**DI-2: Impact does not compound across operations**
The report claims the small per-operation error compounds, but each operation independently rounds, and the error is absorbed or corrected by subsequent operations. There is no accumulation mechanism.

**DI-3: Loss is below minimum transferable/withdrawable amount**
The theoretical loss is smaller than the protocol's minimum transfer or withdrawal threshold. The affected user cannot even realize the loss because it's below the minimum operation size.

**DI-4: Precision loss is within acceptable tolerance for the domain**
The precision loss is within the standard tolerance for DeFi operations (typically <0.01%). Financial protocols inherently have rounding, and this level of imprecision is expected and acceptable.

---

## TIMING_IMPOSSIBLE

**TI-1: Attack requires same-block multi-tx ordering control**
The attack requires multiple transactions in the same block in a specific order. Without being the block proposer or paying extreme MEV bribes, an attacker cannot guarantee transaction ordering.

**TI-2: MEV protection makes frontrunning impractical**
The protocol uses private mempool submission (Flashbots Protect, MEV Blocker), commit-reveal, or other MEV protection that prevents the frontrunning/sandwiching described in the report.

**TI-3: Timelock delay exceeds the attack window**
The attack requires a parameter change to take effect before a condition expires, but the protocol's timelock delay is longer than the attack window, giving defenders time to respond.

**TI-4: Oracle update frequency prevents stale price exploitation**
The report assumes the oracle price can be stale, but the oracle's update frequency (heartbeat) and deviation threshold ensure prices are fresh enough that the described manipulation window doesn't exist in practice.

---

## SPEC_COMPLIANT

**SC-1: Behavior matches EIP/ERC specification exactly**
The described behavior is mandated by the relevant EIP or ERC standard. Changing it would make the contract non-compliant. The report identifies spec-compliant behavior as a vulnerability.

**SC-2: Documentation explicitly describes this behavior**
The protocol's whitepaper, docs, or NatSpec comments explicitly describe the behavior the report flags. Users are expected to understand this behavior before interacting.

**SC-3: Behavior is standard across all major implementations**
The described pattern (e.g., first-depositor share inflation in ERC4626, rounding direction in division) is standard across all major DeFi protocols. It's a known property of the mathematical model, not a bug.

---

## INCORRECT_MATH

**IM-1: Report's profit calculation ignores fees**
The report's claimed profit/loss calculation omits transaction fees, protocol fees, swap fees, or other costs that significantly reduce or eliminate the described impact.

**IM-2: Report assumes linear scaling but function is capped**
The report extrapolates impact linearly, but the actual function has a cap, ceiling, or diminishing returns that bound the maximum impact well below the claimed amount.

**IM-3: Report uses wrong decimal/precision in calculation**
The proof-of-concept or impact calculation uses incorrect decimal places, token precision, or unit conversion, producing inflated impact numbers.

**IM-4: Report conflates theoretical maximum with realistic impact**
The report presents the absolute worst-case mathematical maximum as the expected impact. Under any realistic market conditions, the actual impact is orders of magnitude smaller.

---

## ALREADY_MITIGATED

**AM-1: Separate function resets the vulnerable state**
Another function in the protocol (e.g., a periodic settlement, rebalance, or sync call) resets the state that the attack depends on. The vulnerable state is transient and self-correcting.

**AM-2: Timelock or governance delay allows defensive response**
The attack requires a governance action or parameter change, but the timelock delay gives monitoring systems and governance participants enough time to detect and counter the attack.

**AM-3: Circuit breaker or rate limit bounds maximum damage**
The protocol has a circuit breaker, rate limiter, or per-transaction cap that bounds the maximum extractable value per period. The report ignores these limits in its impact calculation.

**AM-4: Monitoring and emergency pause can halt the attack**
The protocol has active monitoring with the ability to pause affected operations. While not a code-level fix, this operational control significantly reduces the practical impact window.

---

## OUT_OF_SCOPE

**OS-1: Affected code is in test/mock/example file only**
The vulnerable code exists only in test helpers, mock contracts, or example files that are never deployed to production. It has zero impact on the live protocol.

**OS-2: Vulnerability is in an external dependency**
The issue is in a third-party library, oracle, or external contract that the protocol integrates but does not control. The protocol uses the dependency correctly according to its API.

**OS-3: Code path is unreachable from any deployed entry point**
The vulnerable function is internal/private and is never called from any public/external function in the deployed contract. Dead code cannot be exploited.

**OS-4: Finding applies to a deprecated or decommissioned component**
The affected contract or function has been deprecated, is being phased out, or is not part of the active protocol. No user funds flow through this code path.
