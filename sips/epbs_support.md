|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Shane Moore    | ePBS (EIP-7732) Support    | Core       | draft               | 2026-04-18 |

## Summary

Describes the SSV spec changes needed to keep SSV operators performing validator duties correctly after ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732), is implemented in Ethereum's consensus layer Gloas fork. Based on the pinned [Gloas consensus-spec snapshot](https://github.com/ethereum/consensus-specs/tree/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas) (`ethereum/consensus-specs@e34dbbb33`, reviewed 2026-06-01).

Validator client related changes via ePBS:
1. earlier slot deadlines
2. attestation handling changes, including preservation of the Gloas `attestation.data.index` semantics
3. the new Payload Timeliness Committee (PTC)
4. the Gloas proposer flow using `produceBlockV4`. The SSV cluster signs the `Gloas.BeaconBlock` in §4 and, on the self-build path, signs `SignedExecutionPayloadEnvelope` as a companion QBFT duty (§6).
5. the new `SignedProposerPreferences` message must be submitted if the node operator wants to be able to select block bids received over p2p

## Motivation

Gloas changes validator duties in ways that break a few current SSV assumptions:

- attestation `index` is no longer safely reconstructible from local validator duty data
- the new PTC duty has a late in-slot deadline
- SSV validators must broadcast `SignedProposerPreferences` or they cannot accept external-builder bids for their slots

## Rationale

Key design choices and why:

- **New `GloasBeaconVote` carries `AttestationDataIndex`.** In Gloas, `AttestationData.Index` is BN-supplied and part of the signed attestation root, so it must travel through QBFT consensus data rather than being reconstructed locally. A dedicated Gloas-only type keeps pre-Gloas `BeaconVote` wire bytes unchanged.
- **PTC is a validator-scoped, non-QBFT runner.** Each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% cutoff; partial signatures group by signing root and reconstruct when one reaches threshold, the same one-round shape as `ProposerPreferences` (§5). A PTC vote is one beacon node's observation, not a value to negotiate, so QBFT would only add round-trips that risk the late-slot deadline (§3).
- **Proposer-preferences is validator-scoped and non-QBFT.** The per-validator `fee_recipient` is configured cluster-side and is cluster-consistent in practice; `target_gas_limit` lives in operator config (ssv-spec default `DefaultGasLimit = 30_000_000` in `types/beacon_types.go`, with runtime overrides; operators in a cluster must agree byte-for-byte on the value used at signing time, same as the existing validator-registration flow). Operators independently derive the full `ProposerPreferences`, partial signatures are grouped by signing root, and reconstruction succeeds only when one signing root reaches threshold. The registration-like one-round partial-sig-and-submit flow from `voluntary_exit.md` fits directly.
- **Block QBFT remains scoped to the `Gloas.BeaconBlock`.** `ProposerConsensusData.data_ssz` carries the block SSZ, matching today's shape. Distributed signing of `SignedExecutionPayloadEnvelope` for the self-build path is covered by a separate companion QBFT duty (§6), keyed by the block QBFT's decided block root.
- **Envelope QBFT uses a blinded envelope shape.** §6's duty runs QBFT over `BlindedExecutionPayloadEnvelope` (`payload` → `payload_root: Root`), whose hash tree root equals the full envelope's. Keeps QBFT messages bounded (~few hundred bytes vs hundreds of KB to ~MB).

## Specification

### 1. Slot Timing Changes

All existing validator duty deadlines shift earlier in the slot. A new PTC deadline is added.

Relevant consensus-spec references:

- [Validator time parameters](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#time-parameters)

| Duty | Pre-ePBS | Post-ePBS (Gloas) |
|------|----------|--------------------|
| Attestation | 1/3 slot (~4s) | 1/4 slot (25%, ~3s) |
| Sync Committee Message | 1/3 slot | 1/4 slot (25%) |
| Aggregation | 2/3 slot (~8s) | 1/2 slot (50%, ~6s) |
| Sync Committee Contribution | 2/3 slot | 1/2 slot (50%) |
| PTC Attestation | - | 3/4 slot (75%, ~9s) |

### 2. Modified Attestation Duty

Relevant consensus-spec references:

- [Validator attestation changes](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#attestation)

#### Consensus-spec change

Under Gloas, the attestation `index` field no longer carries the beacon committee index:

- if attesting to a same-slot block, set `index = 0`
- otherwise, for a non-same-slot attestation:
  - `index = 0` means payload `EMPTY`
  - `index = 1` means payload `FULL`

This value is fork-choice dependent and is supplied by the beacon node in the `AttestationData` returned to the validator client.

#### Why the current SSV model is insufficient

SSV currently omits `AttestationData.Index` from `BeaconVote` and fills it locally at reconstruction (`0` on the current Electra/Fulu path, duty-derived earlier). Post-Gloas this is unsafe: `Index` is BN-supplied and part of the signed attestation root, so if SSV drops it from consensus data, operators can agree on the same `block_root/source/target` and still sign the wrong root.

#### Required change

A new `GloasBeaconVote` struct mirrors `BeaconVote` plus an `AttestationDataIndex` field, matching the `phase0.CommitteeIndex` (a `uint64` alias) type of `AttestationData.Index` in consensus specs so reconstruction is a direct field assignment. Gloas-era QBFT instances decide on `GloasBeaconVote`; pre-Gloas instances continue to decide on the existing `BeaconVote`, whose SSZ layout is unchanged. A separate type (rather than extending `BeaconVote` in place) is the cleanest way to make pre-Gloas and Gloas wire bytes mutually-rejecting on length mismatch, since SSZ derives do not support fork-conditional fields. The restricted Gloas value space (`0` = `EMPTY`, `1` = `FULL` for non-same-slot attestations; `0` for same-slot) is enforced in the value check below, not at the type level.

After the Gloas fork epoch has activated on all networks and pre-Gloas slots are no longer reachable in normal operation, a follow-up SIP can retire `BeaconVote` and rename `GloasBeaconVote` back to `BeaconVote`.

```go
// Existing (ssv-spec types/consensus_data.go); unchanged
type BeaconVote struct {
    BlockRoot phase0.Root `ssz-size:"32"`
    Source    *phase0.Checkpoint
    Target    *phase0.Checkpoint
}

// New (Gloas only)
type GloasBeaconVote struct {
    BlockRoot            phase0.Root        `ssz-size:"32"`
    Source               *phase0.Checkpoint
    Target               *phase0.Checkpoint
    AttestationDataIndex phase0.CommitteeIndex // copied from AttestationData.Index
}
```

#### Value check

A new `GloasBeaconVoteValueCheckF()` mirrors today's `BeaconVoteValueCheckF()` and additionally:

- rejects `AttestationDataIndex` values other than `0` or `1`;
- builds the `AttestationData` passed to `IsAttestationSlashable` using the decided `AttestationDataIndex` rather than the existing `math.MaxUint64` sentinel, so the Gloas double-vote predicate trips correctly when an operator is asked to sign both `index=0` and `index=1` for the same `(source, target, slot)`.

The existing sentinel is in place because pre-Gloas consensus data carries no `CommitteeIndex`; `math.MaxUint64` keeps `IsAttestationSlashable` from flagging legitimate same-`(source, target, slot, BlockRoot)` attestations as double-votes.

The [Gloas same-slot rule](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#attestation) (`block.slot == data.slot ⇒ data.index = 0`) is not enforced locally: the cluster has only the QBFT-decided `BlockRoot` and trusts `AttestationDataIndex` from the leader. A single bad same-slot `index=1` is rejected by the ethereum network and ignored on chain but is not slashable, while cross-`index` equivocation over the same `(source, target, slot, BlockRoot)` is still caught by `IsAttestationSlashable` per the previous bullet.

Pre-Gloas slots continue to run `BeaconVoteValueCheckF()` unchanged.

#### Implementation note: aggregation path

The `BNRoleAggregator` duty (handled by the aggregator-committee runner) fetches aggregated attestations from the Beacon API's aggregate-attestation endpoint with `attestation_data_root` as an input. Implementations must compute that root from the BN-supplied `AttestationData` (including its Gloas `index`).

### 3. New Duty: Payload Timeliness Committee (PTC) Attestation

Relevant consensus-spec references:

- [Validator payload timeliness attestation flow](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#payload-timeliness-attestation)
- [Beacon-chain payload attestation containers](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/beacon-chain.md#payloadattestationdata)
- [Fork-choice payload attestation deadline](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/fork-choice.md#new-get_payload_attestation_due_ms)

PTC is a per-slot consensus-layer-selected set of validators that attests to payload and blob availability for the slot's beacon block.

Each validator signs a `PayloadAttestationData` object carrying `beacon_block_root`, `slot`, `payload_present`, and `blob_data_available`, then submits a validator-specific `PayloadAttestationMessage(validator_index, data, signature)` to the beacon node.

At the start of each epoch, SSV should fetch PTC duties for the next epoch and refresh them on duty-dependent-root changes. Because PTC duty responses may be sparse and incomplete, a changed duty-dependent root for an epoch should replace the cached duties for that epoch rather than being merged.

Two distinct deadlines bound the runner, both nominally at 75% of the slot. `PAYLOAD_DUE_BPS = 75%` is the validator-side observation cutoff: `payload_present` is the predicate "a `SignedExecutionPayloadEnvelope` for `beacon_block_root` was seen before the cutoff" (a first-seen-time test, not a "present now" query), so an envelope arriving after the cutoff does not flip it to `True`. `PAYLOAD_ATTESTATION_DUE_BPS = 75%` is the consensus-spec-recommended broadcast time, a soft target leaving ~25% of the slot for propagation to the next slot's proposer. The hard deadline is slot end (100%): a slot-N payload attestation is accepted on the wire only while `data.slot == current_slot` and is consulted by fork choice only in slot N+1, so nothing consumes a slot-N vote during the [75%, 100%] window. Each operator therefore evaluates `PayloadAttestationData` (`payload_present`, `blob_data_available`, `beacon_block_root`) from its beacon node at the 75% cutoff and runs its partial-signature round in the otherwise-free [75%, 100%] window. Broadcast may complete after 75% and still propagate; past slot end the message is dropped (gossip IGNORE and fork-choice wire REJECT when `data.slot != current_slot`), and each missed vote chips at the `PTC_SIZE/2` threshold that governs whether fork choice extends the payload.

The beacon node also emits SSE events a runner may consume as a push trigger for when to evaluate its observation, instead of polling toward the cutoff: `execution_payload_gossip` and `execution_payload` fire when a `SignedExecutionPayloadEnvelope` passes `execution_payload`-topic gossip validation and when it is imported into fork choice, and `execution_payload_available` fires once the node has verified the payload and blobs are available and ready for payload attestation ([beacon-APIs event stream](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/eventstream/index.yaml)). These are timing signals only: `payload_present` and `blob_data_available` still come from the BN-computed `PayloadAttestationData` at fetch time, bounded by the same cutoff.

There is no pre-consensus phase and no QBFT round. Each operator evaluates `PayloadAttestationData` from its own beacon node at the 75% cutoff and signs that observation directly; there is no leader and no negotiated value. Because a per-validator BLS signature reconstructs only from partial signatures over byte-identical data, each operator pins its 75% `PayloadAttestationData` snapshot (with `slot` taken from the duty) and validates and aggregates incoming partial signatures against exactly that signing root.

An operator that has seen no beacon block for the slot abstains (submits nothing), matching the Gloas validator spec. Otherwise, for each of its local PTC-assigned validators it produces one partial signature over the full `PayloadAttestationData` under `DOMAIN_PTC_ATTESTER` (domain epoch = `compute_epoch_at_slot(duty.slot)`), because each `PayloadAttestationMessage` on the wire ships a validator-specific signature verified against that validator's pubkey. All partial signatures broadcast together in a single `PartialSignatureMessages` container with `Type = PTCAttesterPartialSig` (the runner role `RolePTCAttester` is the dispatch discriminator). Each operator accumulates peers' partial signatures over its own frozen root; when signatures over identical `PayloadAttestationData` reach the reconstruction threshold, it BLS-aggregates and submits one `PayloadAttestationMessage(validator_index, data, signature)` per validator to the beacon node, inside the [75%, 100%] window. Operators on a minority observation never reach threshold and contribute nothing, a non-slashable silent miss (see Security Considerations). The cluster therefore emits, per validator, the observation a threshold of its operators converged on, rather than a single leader's.

The False-vote / missed-vote equivalence holds for `payload_present` only: `blob_data_available = False` votes are additionally counted in `should_build_on_full` via `payload_data_availability(..., available=False)`, so the cluster's observation carries slightly different fork-choice weight across the two boolean fields.

There is no QBFT value check: there is no leader-proposed value to validate, since each operator signs only the observation it made (one that saw no block abstains, as above). The off-slot-root concern that a leader-decided root would raise does not arise, because each operator signs the block it observed for `duty.slot`, on-slot by construction. PTC attestations are not in the beacon chain slashing predicate, so no slashability call is required.

This SIP adds a new beacon role `BNRolePTCAttester`, a matching runner role `RolePTCAttester`, and a new `PartialSigMsgType` `PTCAttesterPartialSig`.

```go
// types/beacon_types.go additions
var (
    // ... existing values ...
    DomainPTCAttester = [4]byte{0x0C, 0x00, 0x00, 0x00}
)

const (
    // ... existing values ...
    BNRolePTCAttester BeaconRole = 7
)

// types/runner_role.go additions 
const (
    // ... existing values ...
    RolePTCAttester RunnerRole = 7
)

// types/partial_sig_message.go additions
const (
    // ... existing values ...
    PTCAttesterPartialSig PartialSigMsgType = 7
)
```

`RunnerRole` values `1` and `3` are reserved for backward-compat decoding of pre-consolidation messages.

`MapDutyToRunnerRole()` must map `BNRolePTCAttester` to `RolePTCAttester`. PTC reuses the existing `ValidatorDuty`, with one runner instance scoped per PTC-assigned validator (keyed by validator pubkey), the same validator-scoped shape as `ProposerPreferences` (§5). A slot's PTC is `PTC_SIZE` (512) seats selected from that slot's beacon committees, so a cluster typically holds zero or one PTC seat in a given slot; per-validator scoping reuses the single-validator signing path, and committee-style bundling would save almost nothing.

### 4. Modified Proposer Duty

Relevant consensus-spec references:

- [Validator block and sidecar proposal flow](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#block-and-sidecar-proposal)

Under Gloas, `produceBlockV4` replaces the pre-Gloas proposer flow; blinded blocks are removed. The beacon node returns `Gloas.BeaconBlock` on the stateful path (and on any external-build response) or `Gloas.BlockContents` on the stateless self-build path ([beacon-APIs PR #580](https://github.com/ethereum/beacon-APIs/pull/580)).

`ProposerConsensusData` is preserved: its struct shape (`Duty`, `Version`, `DataSSZ []byte`) is unchanged. `DataSSZ` carries the SSZ-encoded `Gloas.BeaconBlock`. The stateless `BlockContents` variant is handled identically: the cluster extracts the block into `DataSSZ` for QBFT; the inline envelope, blobs, and KZG proofs are handled by §6.

Although the struct shape is unchanged, [`ProposerConsensusData.GetBlockData()`](https://github.com/ssvlabs/ssv-spec/blob/85ee4f32e4fc22bae8aacf837153aab3dcd6620b/types/consensus_data.go#L175-L237)'s per-version switch (Capella → Fulu today) needs a new `DataVersionGloas` arm that unmarshals `DataSSZ` as `Gloas.BeaconBlock`.

Pre-consensus RANDAO flow is unchanged. Post-consensus is unchanged: each operator's `PostConsensusPartialSig` packet carries one `PartialSignatureMessage` over the block root under `DOMAIN_BEACON_PROPOSER`. Publish the signed block via the existing beacon API.

**Envelope signing.** Under Gloas, the validator signs `SignedExecutionPayloadEnvelope` only in the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD`, per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)); in the external-build path the builder signs and publishes its own envelope. Distributed signing of `SignedExecutionPayloadEnvelope` for the self-build path is specified in §6.

### 5. Proposer Preferences Duty

Relevant consensus-spec references:

- [Broadcasting SignedProposerPreferences](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/validator.md#broadcasting-signedproposerpreferences)
- [`SignedProposerPreferences` container](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#new-proposerpreferences)
- [`proposer_preferences` gossip topic](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#proposer_preferences)
- [`execution_payload_bid` gossip validation](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#execution_payload_bid)

Under Gloas, each proposer broadcasts `SignedProposerPreferences` on the `proposer_preferences` p2p topic for future proposal slots within the proposer lookahead (the current epoch up to `MIN_SEED_LOOKAHEAD` epochs ahead). The signed `ProposerPreferences` carries `dependent_root`, `proposal_slot`, `validator_index`, `fee_recipient`, and `target_gas_limit`. `dependent_root` pins the proposer-lookahead epoch's seed via `get_proposer_dependent_root(state, epoch)`; operators populate it from the `dependent_root` returned by [`GET /eth/v2/validator/duties/proposer/{epoch}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/duties/proposer.v2.yaml) for the proposal-slot's epoch. Builders listen to this topic and use a proposer's preferences to construct `execution_payload_bid` objects for that proposer's slots. This replaces the pre-Gloas out-of-band relay-registration mechanism, which is gone along with blinded blocks.

Gossip enforces the handshake at the `execution_payload_bid` topic: each bid requires a matching `SignedProposerPreferences` for its `(proposal_slot, dependent_root)` (otherwise IGNORE'd, not forwarded). The bid `fee_recipient` must match the preference (mismatch is REJECT'd), and the bid `gas_limit` must be EIP-1559-compatible with the proposer's `target_gas_limit` via `is_gas_limit_target_compatible` (incompatible is IGNORE'd). Without this duty broadcast, bids for the validator's slots don't propagate across the network, leaving the BN with no trustless external builder options to return.

The flow matches the existing `ValidatorRegistration` / `VoluntaryExit` shape: validator-scoped, non-QBFT, one round of partial signatures, reconstruct, submit. Each operator signs `ProposerPreferences` under `DOMAIN_PROPOSER_PREFERENCES` with the validator's BLS share key. `fee_recipient` is configured cluster-side (cluster-consistent in practice); `target_gas_limit` lives in operator config (ssv-spec default `DefaultGasLimit = 30_000_000`, with runtime overrides; operators in a cluster must agree byte-for-byte at signing time); `dependent_root` is observed per-operator from the BN's v2 proposer-duties endpoint. Each operator's choice of these three inputs determines its signing root; divergence splits signing roots, and reconstruction succeeds only when one root reaches threshold (same shape as `ValidatorRegistration` today). `fee_recipient` is cluster-consistent in practice; the practical divergence risks are `target_gas_limit` (operator config) and `dependent_root` (observation timing around reorgs and epoch boundaries).

Trigger: at each epoch boundary, and on duty-dependent-root changes for any epoch in the proposer lookahead, iterate local validators and emit one duty per slot returned by `get_upcoming_proposal_slots(state, validator_index)`. In the `MIN_SEED_LOOKAHEAD` epochs immediately before `GLOAS_FORK_EPOCH`, this SIP requires operators to emit preferences for any local-validator proposal slots in the first Gloas epoch. The semantics of `get_upcoming_proposal_slots` plus the gossip rule that `preferences.proposal_slot` must be within the proposer lookahead leave no other emission window for those slots; pre-fork emission also gives builders enough time to receive preferences and produce bids for early Gloas slots, aligning with the spec's *"Proposers SHOULD broadcast their preferences in the epoch before the fork"* recommendation in `p2p-interface.md`. The `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple; emission-timing implications are covered in Security Considerations. If the proposer lookahead for an epoch changes, or `dependent_root` changes for an epoch already in the lookahead, cached duties for that epoch are replaced and a new preference is emitted. Because `dependent_root` is part of the gossip identity above, the new preference is a distinct tuple rather than a replacement of the prior one.

This SIP adds a new beacon role `BNRoleProposerPreferences`, a matching runner role `RoleProposerPreferences`, and a new `PartialSigMsgType` `ProposerPreferencesPartialSig`.

```go
// types/beacon_types.go additions
var (
    // ... existing values ...
    DomainProposerPreferences = [4]byte{0x0D, 0x00, 0x00, 0x00}
)

const (
    // ... existing values
    BNRoleProposerPreferences BeaconRole = 8
)

// types/runner_role.go additions
const (
    // ... existing values
    RoleProposerPreferences RunnerRole = 8
)

// types/partial_sig_message.go additions
const (
    // ... existing values
    ProposerPreferencesPartialSig PartialSigMsgType = 8
)
```

`MapDutyToRunnerRole()` must map `BNRoleProposerPreferences` to `RoleProposerPreferences`.

### 6. New Duty: Envelope Signing (Self-Build Path)

Relevant consensus-spec references:

- [`ExecutionPayloadEnvelope` container](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/beacon-chain.md#executionpayloadenvelope)
- [`execution_payload` gossip topic (carries `SignedExecutionPayloadEnvelope`)](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#execution_payload)
- [`POST /eth/v1/beacon/execution_payload_envelopes` endpoint](https://github.com/ethereum/beacon-APIs/pull/580) (beacon-APIs PR #580, not yet merged)

On the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD` per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)), the proposer signs `SignedExecutionPayloadEnvelope` after block publication. The SSV cluster runs a second QBFT round to produce this signature.

#### Blinded envelope type

To bound QBFT message size, the cluster runs QBFT over a blinded form that substitutes `payload` with `payload_root: Root = hash_tree_root(payload)`. By SSZ Container positional merkleization, `hash_tree_root(BlindedExecutionPayloadEnvelope) == hash_tree_root(ExecutionPayloadEnvelope)`, so a BLS sig over the blinded signing root is valid for the full envelope.

```go
type BlindedExecutionPayloadEnvelope struct {
    PayloadRoot           phase0.Root // == hash_tree_root(envelope.payload)
    ExecutionRequests     electra.ExecutionRequests
    BuilderIndex          uint64
    BeaconBlockRoot       phase0.Root
    ParentBeaconBlockRoot phase0.Root
}
```

#### Trigger and envelope source

Fires on the self-build path only, after the §4 block is signed and published. No pre-consensus phase. Envelope source by self-build variant: stateless self-build returns the envelope inline in `BlockContents`; stateful self-build requires a `GET /eth/v1/validator/execution_payload_envelopes/{slot}/{beacon_block_root}` to the same BN that served the §4 block (envelope held server-side keyed by that call).

The duty must target publishing the signed envelope before `get_payload_due_ms()` (the `PAYLOAD_DUE_BPS` 75% cutoff, §3), with margin for gossip to reach PTC beacon nodes before then. The §4 block is already out by the attestation deadline (§1), so the round has the intervening block-to-payload gap to work in. An envelope that misses the cutoff makes honest PTC validators vote `payload_present = False`, which can leave the next slot building on the empty parent (§3); this is the bounded missed-envelope degradation in Security Considerations.

#### Roles and constants

```go
// types/beacon_types.go
var DomainBeaconBuilder = [4]byte{0x0B, 0x00, 0x00, 0x00}
const BNRoleEnvelopeBuilder BeaconRole = 9

// types/runner_role.go
const RoleEnvelopeBuilder RunnerRole = 9

type EnvelopeConsensusData struct {
    Duty    ValidatorDuty
    Version spec.DataVersion
    DataSSZ []byte // SSZ-encoded BlindedExecutionPayloadEnvelope
}
```

`MapDutyToRunnerRole()` must map `BNRoleEnvelopeBuilder` to `RoleEnvelopeBuilder`. Post-consensus reuses `PostConsensusPartialSig`; the runner role discriminates routing.

#### QBFT proposal

Each operator constructs `BlindedExecutionPayloadEnvelope` from its local BN's envelope (`PayloadRoot = hash_tree_root(envelope.payload)`, other fields verbatim) and proposes the SSZ-encoded form in `EnvelopeConsensusData.DataSSZ`. Only an operator whose BN built the §4-decided block holds an envelope with a matching `BeaconBlockRoot` and a `PayloadRoot` backed by real full bytes, so only such an operator originates a publishable value in the first round. The QBFT value is the blinded envelope, so under round-change a later-round leader can re-propose that justified value without holding the full bytes; the decided value, and which operator can publish it, are independent of who leads the deciding round (publication is by content-match, see Publication).

#### Value check

A new `EnvelopeValueCheckF()` accepts the decided blinded envelope if all of:

- SSZ decode of `DataSSZ` into `BlindedExecutionPayloadEnvelope` succeeds;
- `EnvelopeConsensusData.Duty.{Slot, ValidatorIndex, PubKey}` match the duty the runner was started with;
- `BuilderIndex == BUILDER_INDEX_SELF_BUILD` (external builders sign their own envelopes; this duty applies only to the self-build path);
- `BeaconBlockRoot` matches the block root decided by the §4 block QBFT for the slot.

No envelope-content validation: `PayloadRoot` (and therefore every constituent field of the underlying `ExecutionPayload` such as `transactions`, `withdrawals`, `state_root`, `block_hash`) is leader-decided and trusted by the cluster. This matches the existing blinded-block trust model in `BNRoleProposer`, where operators do no field-level validation against their local BN view (see Security Considerations).

#### Post-consensus

Operators sign the decided `BlindedExecutionPayloadEnvelope`'s signing root under `DOMAIN_BEACON_BUILDER` (`0x0B000000`); by SSZ root-equivalence this is the full envelope's signing root. Partial sigs broadcast as `PostConsensusPartialSig`.

#### Publication

Each operator's BN built a different full envelope; only the operator whose blinded form matched the QBFT decision holds the matching full bytes. That operator reconstructs the signature and POSTs the signed envelope to `/eth/v1/beacon/execution_payload_envelopes` ([beacon-APIs PR #580](https://github.com/ethereum/beacon-APIs/pull/580)); the body depends on the self-build variant (§6 Trigger). For stateless self-build the envelope arrived inline in `BlockContents`, so that operator also holds the matching blobs and KZG proofs and publishes `SignedExecutionPayloadEnvelopeContents` (the `SignedExecutionPayloadEnvelope` plus `blobs` and `kzg_proofs`). For stateful self-build the BN that served the envelope already cached the side data, so a bare `SignedExecutionPayloadEnvelope(full_envelope, reconstructed_sig)` suffices. Other operators complete without publishing.

## Security Considerations

### `GloasBeaconVoteValueCheckF` must include `AttestationDataIndex` in slashability checks

Under Gloas, `AttestationData.Index` is part of the attestation data root and therefore part of the double-vote slashing predicate. `GloasBeaconVoteValueCheckF` must reconstruct the full Gloas `AttestationData` with `Index` from the decided `GloasBeaconVote.AttestationDataIndex` before calling `IsAttestationSlashable`; otherwise an operator could sign `index=0` and `index=1` for the same `(source, target)` in the same slot without the predicate tripping.

### Gloas `AttestationData.Index` is trusted from the QBFT leader

The Gloas `AttestationData.Index` value check (§2) does not require the QBFT-decided value to match each operator's local BN view. Requiring local agreement would fail QBFT rounds whenever operators observe fork-choice state at slightly different times around the deadline, a normal gossip-lag scenario. Accepted tradeoff: a malicious QBFT leader can push a value contrary to the cluster's majority BN observation. This matches existing ssv-spec treatment of `BeaconVote.BlockRoot`, which is trusted from the leader because BNs legitimately diverge on fork-choice head.

### PTC reconstruction is honest-convergence, not consensus

PTC runs no QBFT (§3): each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% cutoff, and a per-validator signature reconstructs only when a threshold of operators converged on byte-identical data. There is no leader to push a value contrary to the cluster's observation, and an operator can only ever vote its own honest observation. The cost is liveness rather than safety: when operators' beacon nodes split across observations near the cutoff (envelope-arrival or head jitter at the `MAXIMUM_GOSSIP_CLOCK_DISPARITY` boundary, or operators on diverged forks), no observation may reach threshold and the cluster's vote for that validator is a silent miss. That miss is non-slashable and its only effect is the foregone contribution to the `PTC_SIZE/2` fork-choice tally, bounded by SSV's PTC seat share. The off-slot-root case that Gloas gossip guards with an IGNORE-level `block.slot == data.slot` check ([`p2p-interface.md`](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#L346-L347)) cannot arise here: each operator signs the block it observed for `duty.slot`.

### Config divergence silently disables trustless external builder bids

`ProposerPreferences` reconstruction requires a quorum of operators to derive the same signing root, which depends on `target_gas_limit` (per-operator config) and `dependent_root` (per-operator BN observation). Divergence on either splits signing roots; if no root reaches threshold, there is no reconstructed signature, no gossip publication, and therefore no matching preference on the `execution_payload_bid` topic; bids for the slot are IGNORE'd by gossip (§5), leaving the BN with no trustless external builder options to return. Same reconstruction failure shape as `ValidatorRegistration` today.

### Too-early `SignedProposerPreferences` publication pins the wrong preference

Because the `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple (§5), reconstructing and publishing a preference before all tuple inputs are final can durably pin a wrong-input preference: a later corrected message for the same tuple is dropped by gossip rather than treated as a replacement. Builders keep using the stale preference, and bids matching the corrected values fail the §5 handshake. Operators must therefore hold publication until `dependent_root`, `fee_recipient`, and `target_gas_limit` are all final for the tuple, and re-emit only when the tuple itself changes (notably when `dependent_root` shifts due to reorg, or `proposer_lookahead` reassigns the validator to a different slot). Distinct from the config-divergence entry above: there, divergence prevents publication; here, premature publication pins the wrong preference more durably than no publication would.

### Late `dependent_root` change near the proposal slot may leave the slot with no matching bid

Late `dependent_root` change tightens the re-emission window. Under non-finality, a deep reorg affecting the end-of-p-2 dependent block forces the proposer to re-emit `SignedProposerPreferences` with the new root; if the re-emission + builder-bid gossip round-trip cannot complete before the proposal deadline, the slot falls through to §6 self-build with a compressed envelope-signing window.

### Matching-envelope operator failure or late publication misses the slot's envelope

The slot's envelope is missed if the operator whose envelope blinds to the §6-decided value (the only one holding the matching full bytes; see §6 Publication) fails before publishing it, or if the envelope-QBFT round completes after `get_payload_due_ms()` so the signed envelope reaches PTC validators too late to observe before the cutoff. In either case PTC records `payload_present = FALSE` (§3); proposer forfeits the payload reward. No worse than the no-envelope-signing baseline for the self-build path.

## Open Questions / Upstream Watchlist

This section is intentionally limited to upstream items that could still change the normative SSV behavior described above. If any of these settle differently, this SIP should be updated.

- `produceBlockV4` shape stabilization: this SIP relies on the current reviewed shape of `produceBlockV4` (`apis/validator/block.v4.yaml`), which has not been merged to `beacon-APIs/master` yet (it lives in [PR #580](https://github.com/ethereum/beacon-APIs/pull/580)). Watch for changes to the response variant discriminator (stateful `BeaconBlock` vs stateless `BlockContents`) and the block submission wrapper shape.
- Validator-facing `SignedProposerPreferences` publication endpoint: the Gloas validator spec expects validators to broadcast preferences to the [`proposer_preferences`](https://github.com/ethereum/consensus-specs/blob/e34dbbb330c14cdd6e62b6f78817d70041abd5b5/specs/gloas/p2p-interface.md#proposer_preferences) gossipsub topic, but `beacon-APIs/master` does not yet expose a validator-facing publication endpoint. §5 is specified against a future `SubmitProposerPreferences(...)` BN abstraction method whose concrete Beacon API shape is TBD.
- Envelope publication and fetch endpoints: §6 specifies `POST /eth/v1/beacon/execution_payload_envelopes` for the self-build envelope-signing duty (publication path) and `GET /eth/v1/validator/execution_payload_envelopes/{slot}/{beacon_block_root}` for the runner-side envelope fetch (retrieval from the same BN that returned the `BlockContents` response). Both live in [beacon-APIs PR #580](https://github.com/ethereum/beacon-APIs/pull/580), which has not been merged to `beacon-APIs/master`. Watch for endpoint path, request/response body, and SSZ encoding changes.
