| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Shane Moore | Early RANDAO Pre-Consensus Emission | Core | draft | [ePBS (EIP-7732) Support (#94)](https://github.com/ssvlabs/SIPs/pull/94) | 2026-07-23 |

[Discussion](https://github.com/ssvlabs/SIPs/discussions/100)

Depends on the ePBS SIP (currently [PR #94](https://github.com/ssvlabs/SIPs/pull/94)); per [SIP-0](sip0.md), this SIP cannot move to last-call before that SIP is approved, and is rejected if it is rejected (unless changed to remove the dependency).

**Summary**

Operators may emit their existing block-proposal RANDAO partial signature up to 2 slots before the proposal slot, so the cluster reconstructs `randao_reveal` before the slot starts instead of inside the post-Gloas ~3s block-production budget. No new duty, role, message kind, domain, topic, or container: the existing Proposer-duty `RandaoPartialSig` message is emitted earlier, stamped with the proposal slot as today. Changes are confined to message-validation timing, ordering, and duty-handling rules plus a bounded receive-side buffer. The rules activate by target proposal slot `S`: they apply when `epoch(S) >= GLOAS_FORK_EPOCH`, even if the permitted emission window for `S` begins before the fork in wall-clock time. The dependency on the ePBS SIP is activation coupling only.

**Motivation**

Gloas moves the attestation deadline to 1/4 slot. RANDAO pre-consensus (sign, gossip, collect 2f+1, reconstruct) is the first serial step of block production and costs a gossip round trip at slot start. The signed object, `SSZUint64(epoch)` under `DOMAIN_RANDAO`, is a pure function of the proposal slot's epoch, so it can be signed and exchanged at any time and a re-org can never invalidate it. Failure modes fall back to today's in-slot exchange, except at a receiver that already IGNOREd the early copy: that share cannot be re-served in-slot (see *Duty assignment*). Addresses [ssv-spec#373](https://github.com/ssvlabs/ssv-spec/issues/373).

**Rationale & Design Goals**

- No new duty: RANDAO always has an in-slot consumer (the Proposer duty), and a separate duty would still need the in-slot pre-consensus as its fallback. This differs from `ProposerPreferences` (defined in the ePBS SIP, [#94](https://github.com/ssvlabs/SIPs/pull/94)), which must reconstruct before the slot and is mutable.
- The 2-slot value is the maximum earliness receivers MUST support, not a gossip-latency requirement. Producers may emit later within that window, and operators need not emit simultaneously. At a participating receiver, the reveal becomes available once it has collected `2f+1` valid shares, so the latest share needed for quorum determines reconstruction time. Different producer schedules remain interoperable, but later schedules reduce the time saved before block proposal. The window also permits a simple slot-driven producer to make an initial attempt in the `S - 2` slot and retry in `S - 1` if the earlier local publication attempt fails. It does not guarantee a second network delivery after successful publication because byte-identical re-publications are gossip-deduplicated. A longer receiver window would widen reveal and stale-duty exposure without an identified need and would require protocol coordination.
- The per-(signer, slot) duplicate limit stays 1. Each logical partial has exactly one valid byte encoding, gossip message identifiers are content-derived, and gossip layers deduplicate before validation, so a second copy either never reaches validation or is the receiver's first copy.
- Because gossip layers never unmark a seen message, an IGNORE of an early partial permanently discards that operator's share for the window. The Unknown-duty retention rule and the dedicated clock tolerance narrow that failure class for honest shares without eliminating it; the residual cases are documented in *Duty assignment* (stale-`Known` reorg), the retention admission rule (refresh-pending losses), and Security Considerations.

**Specification**

Notation: `S` is a proposal slot; `slot_start(s)` and `epoch(s)` as usual; for a non-genesis Gloas activation, `F` is the first slot of `GLOAS_FORK_EPOCH`; `SLOT_DURATION` is the network's beacon-chain seconds-per-slot (unchanged at Gloas, which alters only intra-slot timing). REJECT penalizes the delivering peer; IGNORE drops without forwarding or penalty.

| Constant | Value |
| -------- | ----- |
| `EARLY_RANDAO_LEAD` | 2 slots |
| `EARLY_RANDAO_CLOCK_TOLERANCE` | 1000 ms |
| `MAX_QUARANTINED_MESSAGES` | 4096 |

**Qualifying message**

A qualifying randao partial MUST satisfy all of:

- Proposer-role `MessageID`; type `RandaoPartialSig`; exactly one `PartialSignatureMessage` entry;
- `slot` = the proposal slot `S`; signed object `SSZUint64(epoch(S))` under `DOMAIN_RANDAO`, domain epoch `epoch(S)`;
- canonical SSZ; deterministic BLS share signature; deterministic RSA (PKCS#1 v1.5) operator signature; exactly one outer signer, equal to the embedded operator ID;
- eligibility predicate: `S >= EARLY_RANDAO_LEAD` and `epoch(S) >= GLOAS_FORK_EPOCH` (the Ethereum Gloas fork epoch, as used by the ePBS SIP, [#94](https://github.com/ssvlabs/SIPs/pull/94)).

The predicate is a pure function of `S`, evaluated identically by producers and receivers. It is not gated by the receiver's current epoch or by whether wall-clock time has reached the Gloas fork. The `S >= EARLY_RANDAO_LEAD` conjunct only keeps the producer's emission window well defined and excludes the first `EARLY_RANDAO_LEAD` slots at genesis.

For a non-genesis activation, `F` and `F + 1` are eligible target slots. Their producer windows begin, by the producer's clock, at `slot_start(F - 2)` and `slot_start(F - 1)`, respectively. There is no activation warm-up: the first Gloas target slot receives the full early window. Because receiver timing includes `EARLY_RANDAO_CLOCK_TOLERANCE`, implementations MUST enable the receiver rules no later than `slot_start(F - 2) - EARLY_RANDAO_CLOCK_TOLERANCE` by the receiver's clock. This is the earliest receiver-local time at which a conforming partial for `F` can pass timing validation.

The SSV fork schedule plays no part, and an emission window that spans an SSV fork activation is fine: every fork-sensitive artifact of a partial signature (gossip topic, domain, role gating) is derived from the message's stamped slot rather than from the moment it was emitted, so such a message is published, validated, and consumed entirely under the fork active at `S`. The same holds for a window spanning the Ethereum fork boundary. Containers violating the canonical form fall to existing structural rules (REJECT); BLS-share validity is not evaluated during message validation (see Message validation). Messages failing the predicate, and all non-randao messages, keep today's validation unchanged.

**Producer behavior**

For a locally known proposer duty at eligible slot `S`, an operator:

- MAY broadcast its qualifying randao partial at any wall-clock time in `[slot_start(S - EARLY_RANDAO_LEAD), slot_start(S))`;
- MUST NOT broadcast it before `slot_start(S - EARLY_RANDAO_LEAD)` by its own clock;
- SHOULD delay emission at least 500 ms past `slot_start(S - EARLY_RANDAO_LEAD)` (assumed maximum pairwise honest clock disparity: 1 s);
- SHOULD still execute the existing in-slot emission at `S` unconditionally; a running origin's identical re-publish is absorbed by its own gossip layer (expected, not an error), and a restarted origin's re-publish aids recovery;
- MAY emit immediately for a duty discovered inside the window; SHOULD emit multiple eligible duties in ascending slot order (correctness does not depend on it).

For `S = F`, the interval begins at `slot_start(F - EARLY_RANDAO_LEAD)`, before wall-clock Gloas activation. Producer scheduling therefore needs to run before `F` to use the full window, but early emission itself remains optional.

Operators that never emit early remain fully conformant.

**Message validation**

Validation of a potential Early RANDAO message begins with the structural and canonical-form checks, which MUST run first; a structurally and canonically valid randao container satisfying the eligibility predicate is an Early RANDAO candidate, and the rules below apply to candidates only (all other messages keep today's validation unchanged). For a candidate, implementations MAY order operator-signature verification and the non-mutating contextual checks below (timing, slot ordering, duty assignment, duplicate limits) according to local denial-of-service policy, and MAY short-circuit on a contextual verdict that neither retains nor accepts the message. Any outcome that retains, accepts, forwards, feeds signature collection, or mutates ordinary validation state MUST first pass operator-signature verification; in particular, a candidate failing it under an Unknown duty view is REJECTed, never retained. If a candidate fails both operator-signature verification and a non-retaining contextual check, either the contextual verdict or the signature-verification REJECT is conformant. Validation state (duplicate counts, signer state, slot high-water marks, and epoch counters) mutates only on acceptance or promotion, never on IGNORE.

Checks are staged: operator-signature verification gates retention, acceptance, forwarding, and state mutation; BLS-share validity gates consumption. A candidate is fully qualifying only once every applicable check has passed; honest producers emit qualifying messages by construction.

*Earliness.* A receiver MUST accept a candidate's timing iff

```text
slot_start(msg.slot) - local_now <= EARLY_RANDAO_LEAD * SLOT_DURATION + EARLY_RANDAO_CLOCK_TOLERANCE
```

inclusive on the accept side; strictly greater is IGNORE. This replaces the generic clock tolerance for this rule only. Lateness is unchanged (existing Proposer-role TTL against `msg.slot`). All other message classes keep their existing allowances.

*Slot ordering.* Candidate randao partials are exempt from the per-(operator, `MessageID`) highest-seen-slot rule in both directions: they MUST NOT be rejected for a slot below the high-water mark, and they MUST NOT advance the high-water mark applied to other message classes on the same `MessageID` (randao shares its `MessageID` with the proposal's consensus and post-consensus traffic). Final acceptance still requires the remaining qualifying-message checks; earliness, lateness, the duplicate limit of 1, and canonical form bound these messages instead.

*Duty assignment.* Tri-state, evaluated per candidate against the receiver's proposer-duty view for `epoch(msg.slot)` and the message's validator:

- Known (a cached view is authoritative for the candidate; see below) and assigned at exactly `msg.slot`: the duty-assignment check passes.
- Known and not assigned: the duty-assignment verdict is IGNORE, never REJECT (receiver views can be stale or re-orged); an independently failing rule may still determine the final verdict under the ordering allowance above.
- Unknown (otherwise): IGNORE; retained only if admitted under the rules below.

A cached view is authoritative for a candidate iff a successfully completed proposer-duty fetch for `epoch(msg.slot)` is cached whose requested coverage includes the message's validator, evaluated against the receiver's current registry state: a fetch that predates the receiver's most recent validator-set change is not authoritative for validators affected by that change. A reorg does not revoke authority: a view made stale by a reorg remains Known, with the loss class described below. A complete-schedule fetch covers every validator, but a response without exactly `SLOTS_PER_EPOCH` distinct in-epoch slots is a fetch failure. A provisional response (one flagged execution-optimistic) installs no view. A registry-filtered fetch covers exactly the validators it requested, even when the filtered result is empty; in particular, a view covering only the operator's own validators MUST NOT yield a not-assigned verdict for claimants outside it.

Receivers SHOULD hold next-epoch proposer duties by the tail of each epoch.

A Known view made stale by a boundary-adjacent reorg (including a provisional next-epoch view) IGNOREs an honest early share without retention. This loss class is intentional: the share is not recoverable (the in-slot re-emission is byte-identical, absorbed by the origin's publish-side deduplication and, from a restarted origin, by the receiver's seen-cache), and the cost is bounded by the round-change path, the same residual class as the round-change case in Security Considerations. Retention deliberately excludes Known-and-not-assigned; retaining it would remove the duty-fetch-failure precondition from the quarantine-exhaustion attack and would require re-evaluating the buffer on every duty-view change. Exposure is transient: receiver views converge within a slot or two of a reorg under normal operation (per-slot re-polling or dependent-root-triggered refetch).

**Unknown-duty retention (IGNORE-AND-RETAIN)**

A candidate whose proposer-duty view is Unknown and that meets the admission conditions below is considered for retention under the occupied-key and capacity rules below; if admitted, it is retained locally before the gossip verdict IGNORE is returned: not forwarded, no penalty, but kept so the seen-cache mark cannot permanently discard it.

- Admission: operator-signature verification and every applicable message-validation rule other than duty assignment have passed, the candidate's duty view is Unknown, and the most recent completed fetch attempt for `epoch(msg.slot)` did not succeed (or none has completed). Attempts are ordered by initiation, so a result arriving for a superseded attempt is discarded for this state; a provisional response counts as a succeeded attempt here, so a long provisional period cannot hold the epoch retention-eligible. A candidate left Unknown by a succeeding fetch (outside its coverage, or affected by a later validator-set change) is IGNOREd without retention: a refresh is already due, and the exposure is the same transient class as the reorg path.
- Key: (operator, validator, `msg.slot`), at most 1 entry. Global capacity `MAX_QUARANTINED_MESSAGES` per node. Sizing: an entry is one signed message (~0.5 KB), bounding memory at ~2 MB. A receiver MAY scope retention to committees it participates in (retained shares are only ever consumed locally; promotion cannot retroactively forward).
- Occupied key, byte-identical content: IGNORE; the original entry and its metadata are kept unchanged.
- Occupied key, distinct bytes: existing two-tier duplicate rule (REJECT from the same delivering peer, IGNORE otherwise); original retained; duplicate-count and ordering state not mutated; quota not recharged.
- Eviction when full: the incoming candidate competes with the stored entries; the greatest `|msg.slot - current_slot|` is evicted first, ties by larger `msg.slot`, then the lexicographically greater validator pubkey (over its canonical 48-byte compressed encoding), then higher operator ID (the key fields make further ties impossible). A candidate that loses the comparison is not admitted (IGNORE).
- Expiry: delete an entry once a newly received copy would fail the lateness rule.
- Promotion: when a newly installed view is authoritative for a retained entry, that entry is decided immediately: an entry assigned at exactly `msg.slot` becomes a promotion candidate, anything else deletes. A candidate is reprocessed exactly as a newly received message, with all stateful rules applied at reprocessing time; it promotes only if that revalidation succeeds and is deleted otherwise (a normally accepted interim copy for the same (signer, slot) makes the candidate the count-limit duplicate, so it deletes).
- Accounting: the retention quota mutates at retention time; ordinary validation state does not mutate on the IGNORE path.

Honest demand cannot approach the cap. At any instant only ~7 slot stamps are admissible (the 3-slot proposer lateness TTL behind, `EARLY_RANDAO_LEAD` ahead, the current slot, and tolerances), and an honest entry additionally requires a real proposer duty in an epoch the receiver has not successfully fetched. That bounds honest entries by one SSV proposal per slot network-wide, times at most 13 committee operators, times the ~7-slot span: under 100 entries. The bound is independent of node size (admission is not scoped to local validators, so demand tracks the network's proposal rate) and of outage duration (the admissible window slides and expiry drains it). Reaching the cap requires fabricated slot stamps from a real signer (see Security Considerations).

Shares that are neither promoted nor normally accepted MUST NOT be fed to signature collection or any downstream reconstruction cache.

**Reconstruction and consumption**

Unchanged. The reconstructed reveal is a per-epoch value: an implementation MAY serve a proposal at slot `Y` from a reconstruction built via shares stamped for slot `X` of the same epoch, provided every contributing share was promoted or normally accepted. Wire messages remain per-slot-stamped.

"Promoted or normally accepted" is evaluated at receipt or promotion time against the receiver's then-current duty view and is never re-evaluated at consumption. The cross-slot case is reachable two ways: a validator with two proposals in the same epoch, and a reorg that moves a proposal from `X` to `Y` after `X`-stamped shares were accepted under the pre-reorg view; the allowance exists so a completed early collection survives the shift. The signed object is identical for `X` and `Y` (`SSZUint64(epoch)`), so reuse has no cryptographic effect; implementations that key collection by signing root exercise it naturally, and implementations that require exact-slot consumption remain conformant.

**Test expectations**

Cross-client vectors MUST cover:

- earliness boundary at exactly `EARLY_RANDAO_LEAD * SLOT_DURATION + EARLY_RANDAO_CLOCK_TOLERANCE` (accept) and beyond (IGNORE);
- Unknown-view retention, promotion, and deletion; Known-unassigned IGNORE;
- the stale-Known reorg path: a share IGNOREd under a pre-reorg Known view is not retained, an identical later copy is dropped by the seen-cache, and a subsequent duty-view update does not resurrect it;
- same-epoch duty move: shares stamped `X` accepted under a duty-at-`X` view, the duty moves to `Y` in the same epoch, and the proposal at `Y` completes (served from the `X` reconstruction where the implementation reuses; via the in-slot exchange otherwise);
- the two-direction ordering exemption with proposals at consecutive slots for one validator;
- occupied-key duplicates: byte-identical IGNORE-keep-original, distinct-bytes two-tier (same-peer REJECT, otherwise IGNORE);
- full-capacity eviction with the incoming candidate winning and losing the comparison;
- Known coverage: a registry-filtered fetch covering every admissible claimant is authoritative, and a covered validator absent from its result is not-assigned; a view scoped to the operator's own validators is not authoritative for an out-of-coverage claimant and must not IGNORE its honest share as Known-unassigned; a claimant registered after the cached fetch takes the Unknown path, without retention, until a refresh covers it; a complete-schedule response without exactly `SLOTS_PER_EPOCH` distinct in-epoch slots establishes no view;
- retention admission: a candidate in a never-fetched epoch retains and one in an epoch whose latest refresh failed retains, while one left Unknown by a succeeding fetch does not; a late failure result from a superseded attempt does not make the epoch retention-eligible; a provisional (execution-optimistic) response ends eligibility while installing no view;
- an operator-authenticated share whose BLS signature is invalid is discarded at consumption and contributes to no reconstructed reveal; reconstruction still succeeds from a valid threshold of honest shares;
- the eligibility predicate at genesis: slots below `EARLY_RANDAO_LEAD` are ineligible even when `GLOAS_FORK_EPOCH` is 0;
- activation gating: `epoch(S)` immediately before `GLOAS_FORK_EPOCH` is ineligible; for a non-genesis `F`, an otherwise qualifying `msg.slot = F` received at receiver-local time `slot_start(F - 2) - EARLY_RANDAO_CLOCK_TOLERANCE` and `msg.slot = F + 1` received at `slot_start(F - 1) - EARLY_RANDAO_CLOCK_TOLERANCE` enter the Early RANDAO validation path even though local wall-clock time is before the fork; an SSV fork scheduled anywhere does not change that;
- multi-fault precedence: a structurally valid message failing both operator-signature verification and a non-retaining contextual check (e.g. invalid signature plus Known-unassigned, or invalid signature plus too-early) may produce either the contextual verdict or REJECT, but is never retained, accepted, forwarded, collected, or state-mutating;
- invalid operator signature with an Unknown duty view: REJECT, never retained;
- restart cases, late mesh join, cross-epoch stamps.

Before mainnet activation guidance, testnet measurement MUST quantify early-share miss incidence at duty start (stratified by emission lead, receiver restarts, partition duration, leader round) and MUST demonstrate round-2 viability under Gloas timing, meaning block publication before the post-Gloas useful-block deadline.

**Security Considerations**

- Deliberately ineligible early emission only starves the attacker's own share (only the signer can produce the bytes). The clock tolerance, Unknown-state retention, and IGNORE-not-REJECT choices narrow the permanent-loss class for honest shares but do not empty it: the stale-`Known` reorg path, the refresh-pending admission exclusion, and the residuals in the bullets below remain.
- After the gossip message-cache window (a few seconds), a stable origin cannot re-deliver a missed early share; receiver restarts and partitions are correlated events. A restart also drops unpromoted retained entries, since this SIP does not mandate persisting the retention buffer. Residual: a live round-1 leader without a reconstruction forces a round change. Mitigations: later emission inside the window, the unconditional in-slot emission (restarted-origin recovery), optional non-normative persistence of received shares, and the mandatory testnet measurement above.
- Early reconstruction widens the third-party reaction window between reveal knowability and block publication from roughly 2-4 s to up to 25 s, enabling adaptive bribery, censorship, or targeted DoS when withholding the proposal favors an adversary (one randao bit per affected slot, compounding across slots). Bounded by the existing single-proposal grinding bound; narrowable operationally via later emission, without a protocol change.
- An attacker meeting the retention preconditions (a committee operator sharing more than `MAX_QUARANTINED_MESSAGES` divided by the ~7 admissible slot stamps ≈ 600 validators with the victim, while the victim has no successful fetch for the stamped epoch; milder outages raise the threshold) can fill the retention capacity and evict legitimate entries; cost is bounded to the evicted shares (round-change path).
- No unauthenticated amplification: messages failing operator-signature verification are never forwarded or retained; authentication cost is unchanged (operator signature at validation, BLS at consumption). An operator-authenticated share whose BLS signature proves invalid at consumption is attributable and MUST be discarded without contributing to a reconstructed reveal. A signer emitting two distinct containers for one (signer, slot) is provably misbehaving and attributable via its operator signature.
- No slashing surface: the randao object is not slashable and is a pure function of public data; early signing changes when it is signed, not what.
