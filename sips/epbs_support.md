|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Shane Moore    | ePBS (EIP-7732) Support    | Core       | draft               | 2026-04-18 |

## Summary

Describes the SSV spec changes needed to keep SSV operators performing validator duties correctly after ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732), is implemented in Ethereum's consensus layer Gloas fork. Based on the pinned [Gloas consensus-spec snapshot](https://github.com/ethereum/consensus-specs/tree/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas) (`ethereum/consensus-specs@801a38e15`, reviewed 2026-07-06).

Validator client related changes via ePBS:
1. earlier slot deadlines
2. attestation handling changes, including preservation of the Gloas `attestation.data.index` semantics
3. the new Payload Timeliness Committee (PTC)
4. the Gloas proposer flow using `produceBlockV4`. The SSV cluster signs the `Gloas.BeaconBlock` in [§4](#4-modified-proposer-duty) and, on the self-build path, signs `SignedExecutionPayloadEnvelope` as a companion QBFT duty ([§6](#6-new-duty-envelope-signing-self-build-path)).
5. the new `SignedProposerPreferences` message must be submitted if the node operator wants to be able to select block bids received over p2p
6. the existing validator-registration duty is deprecated at the Gloas fork, its purpose replaced by `SignedProposerPreferences` ([§5](#5-proposer-preferences-duty))

## Motivation

Gloas changes validator duties in ways that break a few current SSV assumptions:

- attestation `index` is no longer safely reconstructible from local validator duty data
- the new PTC duty has a late in-slot deadline
- SSV validators must broadcast `SignedProposerPreferences` or they cannot accept builder bids for their slots

## Rationale

Key design choices and why:

- **`BeaconVote` gains `AttestationDataIndex` at the Gloas fork.** In Gloas, `AttestationData.Index` is BN-supplied and part of the signed attestation root, so it must travel through QBFT consensus data rather than being reconstructed locally. The pre-Gloas encoding stays frozen for pre-Gloas slots, so the two are mutually rejecting on length.
- **PTC is a validator-scoped, non-QBFT runner.** Each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% broadcast mark, validates incoming partial signatures against its own derived signing root, and reconstructs when that root reaches threshold, the same one-round shape as `ProposerPreferences` ([§5](#5-proposer-preferences-duty)). A PTC vote is one beacon node's observation, not a value to negotiate, so QBFT would only add round-trips that risk the late-slot deadline ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)).
- **Proposer-preferences is validator-scoped and non-QBFT.** The per-validator `fee_recipient` is configured cluster-side and is cluster-consistent in practice; `target_gas_limit` is operator-configured, with a client default when unset; operators in a cluster must agree byte-for-byte on the value used at signing time, the same convergence requirement as the existing validator-registration flow. Operators independently derive the full `ProposerPreferences` and validate incoming partial signatures against their own derived signing root; reconstruction succeeds only when a quorum of operators converge on one signing root. The registration-like one-round partial-sig-and-submit flow from `voluntary_exit.md` fits directly.
- **Block QBFT remains scoped to the `Gloas.BeaconBlock`.** `ProposerConsensusData.data_ssz` carries the block SSZ, matching today's shape. Distributed signing of `SignedExecutionPayloadEnvelope` for the self-build path is covered by a separate companion QBFT duty ([§6](#6-new-duty-envelope-signing-self-build-path)), keyed by the block QBFT's decided block root.
- **Envelope QBFT uses a blinded envelope shape.** [§6](#6-new-duty-envelope-signing-self-build-path)'s duty runs QBFT over `BlindedExecutionPayloadEnvelope` (`payload` → `payload_root: Root`), whose hash tree root equals the full envelope's. Keeps QBFT messages bounded (~few hundred bytes vs hundreds of KB to ~MB).

## Specification

### 1. Slot Timing Changes

All existing validator duty deadlines shift earlier in the slot. A new PTC deadline is added.

Relevant consensus-spec references:

- [Validator time parameters](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#time-parameters)

| Duty | Pre-ePBS | Post-ePBS (Gloas) |
|------|----------|--------------------|
| Attestation | 1/3 slot (~4s) | 1/4 slot (25%, ~3s) |
| Sync Committee Message | 1/3 slot | 1/4 slot (25%) |
| Aggregation | 2/3 slot (~8s) | 1/2 slot (50%, ~6s) |
| Sync Committee Contribution | 2/3 slot | 1/2 slot (50%) |
| PTC Attestation | - | 3/4 slot (75%, ~9s) |
| Envelope reveal (self-build, [§6](#6-new-duty-envelope-signing-self-build-path)) | - | 1/2 slot (50%, ~6s) |

### 2. Modified Attestation Duty

Relevant consensus-spec references:

- [Validator attestation changes](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#attestation)

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

At the Gloas fork, `BeaconVote` is extended with an `AttestationDataIndex` field (`phase0.CommitteeIndex`, a `uint64` alias matching `AttestationData.Index`, so reconstruction is a direct field assignment). The pre-Gloas 3-field encoding stays frozen for pre-Gloas slots; decoders select by fork, and the two encodings are mutually rejecting on length (fixed-size SSZ, 112 vs 120 bytes). Implementations MAY realize the transition as two concrete types. The restricted Gloas value space (`0` = `EMPTY`, `1` = `FULL` for non-same-slot attestations; `0` for same-slot) is enforced in the value check below, not at the type level.

```go
// Pre-Gloas encoding (ssv-spec types/consensus_data.go); frozen for pre-Gloas slots
type BeaconVote struct {
    BlockRoot phase0.Root `ssz-size:"32"`
    Source    *phase0.Checkpoint
    Target    *phase0.Checkpoint
}

// Gloas encoding; implementations may keep a distinct type through the transition
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

The [Gloas same-slot rule](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#attestation) (`block.slot == data.slot ⇒ data.index = 0`) is not enforced locally: the cluster has only the QBFT-decided `BlockRoot` and trusts `AttestationDataIndex` from the leader. A single bad same-slot `index=1` is rejected by the ethereum network and ignored on chain but is not slashable, while cross-`index` equivocation over the same `(source, target, slot, BlockRoot)` is still caught by `IsAttestationSlashable` per the previous bullet.

Pre-Gloas slots continue to run `BeaconVoteValueCheckF()` unchanged.

#### Implementation note: aggregation path

The `BNRoleAggregator` duty (handled by the aggregator-committee runner) fetches aggregated attestations from the Beacon API's aggregate-attestation endpoint with `attestation_data_root` as an input. Implementations must compute that root over the full Gloas `AttestationData` reconstructed from the decided `GloasBeaconVote` (including `AttestationDataIndex`), rather than from a locally reconstructed pre-Gloas shape; a root computed without the decided `index` matches no aggregate.

`AggregatorCommitteeConsensusData.Version` is part of the QBFT-decided value, so operators must stamp it identically: use `DataVersionGloas` at Gloas slots (as [§4](#4-modified-proposer-duty) does for `ProposerConsensusData`). The aggregate stays Electra-shaped; Gloas changes only `AttestationData.Index` semantics, not the aggregate container.

### 3. New Duty: Payload Timeliness Committee (PTC) Attestation

Relevant consensus-spec references:

- [Validator payload timeliness attestation flow](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#payload-timeliness-attestation)
- [Beacon-chain payload attestation containers](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/beacon-chain.md#payloadattestationdata)
- [Fork-choice payload attestation deadline](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/fork-choice.md#new-get_payload_attestation_due_ms)

PTC is a per-slot consensus-layer-selected set of validators that attests to payload and blob availability for the slot's beacon block.

Each validator signs a `PayloadAttestationData` object carrying `beacon_block_root`, `slot`, `payload_present`, and `blob_data_available`, then submits a validator-specific `PayloadAttestationMessage(validator_index, data, signature)` to the beacon node via [`POST /eth/v1/beacon/pool/payload_attestations`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/beacon/pool/payload_attestations.yaml).

At the start of each epoch, SSV should fetch PTC duties for the current and next epoch via [`POST /eth/v1/validator/duties/ptc/{epoch}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/duties/ptc.yaml) (the endpoint serves at most one epoch ahead; the current-epoch fetch covers operators starting mid-epoch) and refresh them on duty-dependent-root changes. On a dependent-root change, the new response is authoritative for that epoch: cached duties for the epoch are replaced rather than merged, since a merge could retain assignments that no longer exist under the new root.

Two distinct deadlines bound the runner. `PAYLOAD_DUE_BPS = 50%` (lowered from 75% by [consensus-specs #5414](https://github.com/ethereum/consensus-specs/pull/5414)) is the payload-side observation cutoff: `payload_present` is the predicate "a `SignedExecutionPayloadEnvelope` for `beacon_block_root` was seen before the 50% mark" (a first-seen-time test, not a "present now" query), so an envelope arriving after the cutoff never flips it to `True` and the predicate is frozen for the rest of the slot. `blob_data_available` has no such upstream cutoff; it is `is_data_available(beacon_block_root)` at whatever time the data is evaluated. `PAYLOAD_ATTESTATION_DUE_BPS = 75%` is the consensus-spec-recommended broadcast time, a soft target leaving ~25% of the slot for propagation to the next slot's proposer. The hard deadline is slot end (100%): a slot-N payload attestation is accepted on the wire only while `data.slot == current_slot` and is consulted by fork choice only in slot N+1, so nothing consumes a slot-N vote during the [75%, 100%] window. Each operator therefore evaluates `PayloadAttestationData` (`payload_present`, `blob_data_available`, `beacon_block_root`) from its beacon node at the 75% broadcast mark, by which point `payload_present` has been frozen for a quarter slot, and runs its partial-signature round in the otherwise-free [75%, 100%] window. Broadcast may complete after 75% and still propagate; past slot end the message is dropped (gossip IGNORE and fork-choice wire REJECT when `data.slot != current_slot`), and each missed vote chips at the `PTC_SIZE/2` threshold that governs whether fork choice extends the payload.

The beacon node also emits SSE events a runner may consume as a push trigger for when to evaluate its observation, instead of polling toward the cutoff: `execution_payload_gossip` and `execution_payload` fire when a `SignedExecutionPayloadEnvelope` passes `execution_payload`-topic gossip validation and when it is imported into fork choice, and `execution_payload_available` fires once the node has verified the payload and blobs are available and ready for payload attestation ([beacon-APIs event stream](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/eventstream/index.yaml)). These are timing signals only: `payload_present` and `blob_data_available` still come from the BN-computed `PayloadAttestationData` fetched via [`GET /eth/v1/validator/payload_attestation_data/{slot}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/payload_attestation_data.yaml).

There is no pre-consensus phase and no QBFT round. Each operator evaluates `PayloadAttestationData` from its own beacon node at the 75% broadcast mark and signs that observation directly; there is no leader and no negotiated value. Because a per-validator BLS signature reconstructs only from partial signatures over byte-identical data, each operator pins its 75% `PayloadAttestationData` snapshot (with `slot` taken from the duty) and validates and aggregates incoming partial signatures against exactly that signing root.

An operator that has seen no beacon block for the slot abstains (submits nothing), matching the Gloas validator spec. Otherwise, the operator's runner for each of its local PTC-assigned validators produces one partial signature over the full `PayloadAttestationData` under `DOMAIN_PTC_ATTESTER` (domain epoch = `compute_epoch_at_slot(duty.slot)`), because each `PayloadAttestationMessage` on the wire ships a validator-specific signature verified against that validator's pubkey. Each runner broadcasts its partial signature in its own `PartialSignatureMessages` container with `Type = PTCAttesterPartialSig` (the runner role `RolePTCAttester` is the dispatch discriminator), one single-validator container per PTC-assigned validator, the same per-validator container shape as `ValidatorRegistration` today. Each operator accumulates peers' partial signatures over its own frozen root; when signatures over identical `PayloadAttestationData` reach the reconstruction threshold, it BLS-aggregates and submits one `PayloadAttestationMessage(validator_index, data, signature)` per validator to the beacon node, inside the [75%, 100%] window. Operators on a minority observation never reach threshold and contribute nothing, a non-slashable silent miss (see Security Considerations). The cluster therefore emits, per validator, the observation a threshold of its operators converged on, rather than a single leader's.

False votes and missed votes are equivalent in the payload-extension tally (`should_extend_payload` counts only `True` votes toward the threshold), but not for the next proposer's parent choice: explicit `False` majorities on either field steer the proposer off the full parent in `should_build_on_full` (via `payload_timeliness(..., timely=False)` and `payload_data_availability(..., available=False)`), weight a missed vote does not carry.

There is no QBFT and therefore no value check. PTC attestations are not in the beacon chain slashing predicate, so no slashability call is required.

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

`MapDutyToRunnerRole()` must map `BNRolePTCAttester` to `RolePTCAttester`. PTC reuses the existing `ValidatorDuty`, with one runner instance scoped per PTC-assigned validator (keyed by validator pubkey), the same validator-scoped shape as `ProposerPreferences` ([§5](#5-proposer-preferences-duty)). A slot's PTC is `PTC_SIZE` (512) seats selected from that slot's beacon committees, so a cluster typically holds zero or one PTC seat in a given slot; per-validator scoping reuses the single-validator signing path, and committee-style bundling would save almost nothing.

### 4. Modified Proposer Duty

Relevant consensus-spec references:

- [Validator block and sidecar proposal flow](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#block-and-sidecar-proposal)

Under Gloas, `produceBlockV4` (`GET /eth/v4/validator/blocks/{slot}`, merged via [beacon-APIs #580](https://github.com/ethereum/beacon-APIs/pull/580)) replaces the pre-Gloas proposer flow; blinded blocks are removed. The beacon node returns `Gloas.BeaconBlock` on the stateful path (and on any external-build response) or `Gloas.BlockContents` on the stateless self-build path. The variant is signaled by the required `execution_payload_included` response field and `Eth-Execution-Payload-Included` header, and driven by the `include_payload` query parameter (`true` requests the stateless `BlockContents` form).

*Note (non-normative).* When to issue the `produceBlockV4` request is the residual proposer timing discretion: a later request lets the beacon node observe more bids and can return a higher-value block. The window is tighter for an SSV cluster than for a solo proposer, since the returned block must still clear QBFT, post-consensus signing, and propagation before the earlier attestation deadline ([§1](#1-slot-timing-changes)), so the request must leave room for them. Operator policy, not a protocol requirement.

`ProposerConsensusData` is preserved: its struct shape (`Duty`, `Version`, `DataSSZ []byte`) is unchanged. `DataSSZ` carries the SSZ-encoded `Gloas.BeaconBlock`. The stateless `BlockContents` variant is handled identically: the cluster extracts the block into `DataSSZ` for QBFT; the inline envelope, blobs, and KZG proofs are handled by [§6](#6-new-duty-envelope-signing-self-build-path).

Although the struct shape is unchanged, [`ProposerConsensusData.GetBlockData()`](https://github.com/ssvlabs/ssv-spec/blob/85ee4f32e4fc22bae8aacf837153aab3dcd6620b/types/consensus_data.go#L175-L237)'s per-version switch (Capella → Fulu today) needs a new `DataVersionGloas` arm that unmarshals `DataSSZ` as `Gloas.BeaconBlock`.

The decided value's `Version` selects how `DataSSZ` is decoded (`Gloas.BeaconBlock` when `Version >= DataVersionGloas`), and `Version` is leader-supplied. An operator MUST reject any decided value whose `Version` does not equal the fork scheduled at `duty.Slot`. Honest proposers always stamp `Version == fork(duty.Slot)`, so this rejects no honest value.

Pre-consensus RANDAO flow is unchanged. Post-consensus is unchanged: each operator's `PostConsensusPartialSig` packet carries one `PartialSignatureMessage` over the block root under `DOMAIN_BEACON_PROPOSER`. Publish the signed block as `Gloas.SignedBeaconBlock` via the existing block-publish endpoint.

**Envelope signing.** Under Gloas, the validator signs `SignedExecutionPayloadEnvelope` only in the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD`, per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)); in the external-build path the builder signs and publishes its own envelope. Distributed signing of `SignedExecutionPayloadEnvelope` for the self-build path is specified in [§6](#6-new-duty-envelope-signing-self-build-path).

### 5. Proposer Preferences Duty

Relevant consensus-spec references:

- [Broadcasting SignedProposerPreferences](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/validator.md#broadcasting-signedproposerpreferences)
- [`SignedProposerPreferences` container](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/p2p-interface.md#new-proposerpreferences)
- [`proposer_preferences` gossip topic](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/p2p-interface.md#new-proposer_preferences)
- [`execution_payload_bid` gossip validation](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/p2p-interface.md#new-execution_payload_bid)

Under Gloas, each proposer broadcasts `SignedProposerPreferences` on the `proposer_preferences` p2p topic for future proposal slots within the proposer lookahead (the current epoch up to `MIN_SEED_LOOKAHEAD` epochs ahead). The signed `ProposerPreferences` carries `dependent_root`, `proposal_slot`, `validator_index`, `fee_recipient`, and `target_gas_limit`. `dependent_root` pins the proposer-lookahead epoch's seed via `get_shuffling_dependent_root(store, root, epoch)`; operators populate it from the `dependent_root` returned by [`GET /eth/v2/validator/duties/proposer/{epoch}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/duties/proposer.v2.yaml) for the proposal-slot's epoch. Builders listen to this topic and use a proposer's preferences to construct `execution_payload_bid` objects for that proposer's slots. This replaces the pre-Gloas out-of-band relay-registration mechanism, which is gone along with blinded blocks; the `ValidatorRegistration` duty is deprecated accordingly (end of this section).

Gossip enforces the handshake at the `execution_payload_bid` topic: each bid requires a matching `SignedProposerPreferences` for its `(proposal_slot, dependent_root)` (otherwise IGNORE'd, not forwarded). The bid `fee_recipient` must match the preference, and the bid `gas_limit` must be EIP-1559-compatible with the proposer's `target_gas_limit` via `is_gas_limit_target_compatible`; both mismatches are IGNORE'd rather than REJECT'd (the fee-recipient check was downgraded from REJECT by [consensus-specs #5429](https://github.com/ethereum/consensus-specs/pull/5429): preferences are not equivocation-checked, so a mismatch is not provably the bidder's fault). Without this duty broadcast, bids for the validator's slots don't propagate across the network, leaving the BN with no trustless builder options to return.

The flow matches the existing `ValidatorRegistration` / `VoluntaryExit` shape: validator-scoped, non-QBFT, one round of partial signatures, reconstruct, submit. Each operator signs `ProposerPreferences` under `DOMAIN_PROPOSER_PREFERENCES` with the validator's BLS share key; the domain epoch is `compute_epoch_at_slot(proposal_slot)` per the spec's `get_signed_proposer_preferences`, so signatures for the pre-fork emission required below are computed under the Gloas fork domain even while the chain is still on the prior fork. `fee_recipient` is configured cluster-side (cluster-consistent in practice); `target_gas_limit` is operator-configured, with a client default when unset; operators in a cluster must agree byte-for-byte at signing time; `dependent_root` is observed per-operator from the BN's v2 proposer-duties endpoint. Each operator's choice of these three inputs determines its signing root, and operators validate incoming partial signatures against their own derived root (the existing `ValidatorRegistration` expected-root validation); divergence splits signing roots, and reconstruction succeeds only when one root reaches threshold. `fee_recipient` is cluster-consistent in practice; the practical divergence risks are `target_gas_limit` (operator config) and `dependent_root` (observation timing around reorgs and epoch boundaries). The reconstructed `SignedProposerPreferences` is submitted via `POST /eth/v1/validator/proposer_preferences` ([beacon-APIs #608](https://github.com/ethereum/beacon-APIs/pull/608)), which stores it and broadcasts it to the `proposer_preferences` topic.

Trigger: at each epoch boundary, and on duty-dependent-root changes for any epoch in the proposer lookahead, iterate local validators and emit one duty per slot returned by `get_upcoming_proposal_slots(state, validator_index)`. In the `MIN_SEED_LOOKAHEAD` epochs immediately before `GLOAS_FORK_EPOCH`, this SIP requires operators to emit preferences for any local-validator proposal slots in the first Gloas epoch. Emission during the first Gloas epoch itself would also be gossip-valid for slots later in that epoch, but slots early in the epoch leave effectively no post-fork emission time, and builders need a validator's preference before they can construct and gossip bids for its slot; pre-fork emission covers both, aligning with the spec's *"Proposers SHOULD broadcast their preferences in the epoch before the fork"* recommendation in `p2p-interface.md`. The `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple; emission-timing implications are covered in Security Considerations. If the proposer lookahead for an epoch changes, or `dependent_root` changes for an epoch already in the lookahead, cached duties for that epoch are replaced and a new preference is emitted. Because `dependent_root` is part of the gossip identity above, the new preference is a distinct tuple rather than a replacement of the prior one. After a validator-set change (local validators registering, shifting indices), emitters SHOULD additionally delay (re-)emission by a couple of slots so committee peers converge on the new set before the one-shot partials go out; [§7](#7-ssv-message-validation) specifies the matching receive-side tolerance that keeps honest partials valid during the convergence window.

The duty's slot, carried in the `PartialSignatureMessages` envelope, is `proposal_slot` itself: one runner per proposal slot, matching ssv-spec's requirement that a partial-signature message's slot equal its duty's slot (`validatePartialSigMsgForSlot`). At emission time that slot is up to the proposer lookahead in the future, and a re-emission for the same slot carries a new signing root; both effects need dedicated message-validation rules, specified in [§7](#7-ssv-message-validation).

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

#### Deprecation of the `ValidatorRegistration` duty

Gloas removes every protocol consumer of `SignedValidatorRegistrationV1`: builders source `fee_recipient` and `target_gas_limit` from the `proposer_preferences` topic instead, the relay path that consumed registrations is gone with blinded blocks ([§4](#4-modified-proposer-duty)), and the Gloas builder-API workstream itself deprecates `ValidatorRegistrationV1` in favor of `ProposerPreferences` ([builder-specs PR #138](https://github.com/ethereum/builder-specs/pull/138), merged 2026-06-11; see also [builder-specs issue #150](https://github.com/ethereum/builder-specs/issues/150)). This SIP therefore deprecates the duty at the fork: operators emit no `BNRoleValidatorRegistration` duties for epochs at or after `GLOAS_FORK_EPOCH`, message validation treats `ValidatorRegistrationPartialSig` messages for Gloas-or-later slots as invalid, and pre-Gloas slots are unchanged. `BNRoleValidatorRegistration`, `RoleValidatorRegistration`, and `ValidatorRegistrationPartialSig` keep their numeric values, reserved for pre-Gloas operation and backward-compat decoding, the same treatment as `RunnerRole` values `1` and `3` ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)). Where a beacon node sources self-build `fee_recipient` / `target_gas_limit` after the fork is BN configuration outside the SSV protocol, and it needs no distributed signature: registration signatures existed to authenticate proposers to untrusted relays (`prepare_beacon_proposer` carries only `fee_recipient` today).

### 6. New Duty: Envelope Signing (Self-Build Path)

Relevant consensus-spec references:

- [`ExecutionPayloadEnvelope` container](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/beacon-chain.md#executionpayloadenvelope)
- [`execution_payload` gossip topic (carries `SignedExecutionPayloadEnvelope`)](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/p2p-interface.md#new-execution_payload)
- [`POST /eth/v1/beacon/execution_payload_envelopes` endpoint](https://github.com/ethereum/beacon-APIs/pull/580)

On the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD` per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)), the proposer signs `SignedExecutionPayloadEnvelope` after block publication. The SSV cluster runs a second QBFT round to produce this signature.

#### Blinded envelope type

To bound QBFT message size, the cluster runs QBFT over a blinded form that substitutes `payload` with `payload_root: Root = hash_tree_root(payload)`, the same shape as beacon-APIs' `Gloas.BlindedExecutionPayloadEnvelope` ([#580](https://github.com/ethereum/beacon-APIs/pull/580)). In SSZ merkleization every field subtree commits to `hash_tree_root(field)`, so substituting `payload` with its root in the same field position preserves the container root: `hash_tree_root(BlindedExecutionPayloadEnvelope) == hash_tree_root(ExecutionPayloadEnvelope)`, and a BLS sig over the blinded signing root is valid for the full envelope. (If EIP-7688 progressive containers land in Gloas, this note and the blinded struct below must merkleize with the progressive shape; see the watchlist.)

```go
type BlindedExecutionPayloadEnvelope struct {
    PayloadRoot           phase0.Root // == hash_tree_root(envelope.payload)
    ExecutionRequests     gloas.ExecutionRequests // Gloas 5-list container: adds builder_deposits and builder_exits (EIP-8282)
    BuilderIndex          uint64
    BeaconBlockRoot       phase0.Root
    ParentBeaconBlockRoot phase0.Root
}
```

#### Trigger and envelope source

Fires on the self-build path only, after the [§4](#4-modified-proposer-duty) block is signed and published. On the external-build path the builder signs and reveals its own `SignedExecutionPayloadEnvelope`, so this duty does not run. No pre-consensus phase. Envelope source by self-build variant: stateless self-build returns the envelope inline in `BlockContents`; stateful self-build requires a `GET /eth/v1/validator/execution_payload_envelopes/{slot}/{beacon_block_root}` to the same BN that served the [§4](#4-modified-proposer-duty) block, which returns the already-blinded `BlindedExecutionPayloadEnvelope` (cached server-side for the current slot only; keying by block root makes the lookup reorg-resistant).

The duty must target publishing the signed envelope before `get_payload_due_ms()` (the `PAYLOAD_DUE_BPS` cutoff, 50% of the slot since [consensus-specs #5414](https://github.com/ethereum/consensus-specs/pull/5414); [§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)), with margin for gossip to reach PTC beacon nodes before then. The [§4](#4-modified-proposer-duty) block is already out by the attestation deadline ([§1](#1-slot-timing-changes), 25%), leaving roughly a quarter slot (~3s on mainnet) for the envelope QBFT round, post-consensus signing, and publication; #5414 halved this window from the earlier 75% cutoff. An envelope that misses the cutoff makes honest PTC validators vote `payload_present = False`, which can leave the next slot building on the empty parent ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)); this is the bounded missed-envelope degradation in Security Considerations.

#### Roles and constants

```go
// types/beacon_types.go
var DomainBeaconBuilder = [4]byte{0x0B, 0x00, 0x00, 0x00}
const BNRoleEnvelopeProposer BeaconRole = 9

// types/runner_role.go
const RoleEnvelopeProposer RunnerRole = 9

type EnvelopeConsensusData struct {
    Duty    ValidatorDuty
    Version spec.DataVersion
    DataSSZ []byte // SSZ-encoded BlindedExecutionPayloadEnvelope
}
```

`MapDutyToRunnerRole()` must map `BNRoleEnvelopeProposer` to `RoleEnvelopeProposer`. Post-consensus reuses `PostConsensusPartialSig`; the runner role discriminates routing.

#### QBFT proposal

On the stateless path each operator blinds its local BN's inline envelope (`PayloadRoot = hash_tree_root(envelope.payload)`, other fields verbatim); on the stateful path it proposes the fetched `BlindedExecutionPayloadEnvelope` as-is. Either way the SSZ-encoded blinded form goes in `EnvelopeConsensusData.DataSSZ`. Only an operator whose BN built the [§4](#4-modified-proposer-duty)-decided block holds (stateless) or can fetch (stateful) an envelope with a matching `BeaconBlockRoot`, so only such an operator originates a publishable value in the first round. The QBFT value is the blinded envelope, so under round-change a later-round leader can re-propose that justified value without holding it locally; the decided value, and which operator can publish it, are independent of who leads the deciding round (publication is by content-match, see Publication).

#### Value check

A new `EnvelopeValueCheckF()` accepts the decided blinded envelope if all of:

- SSZ decode of `DataSSZ` into `BlindedExecutionPayloadEnvelope` succeeds;
- `EnvelopeConsensusData.Duty.{Slot, ValidatorIndex, PubKey}` match the duty the runner was started with;
- `BuilderIndex == BUILDER_INDEX_SELF_BUILD` (external builders sign their own envelopes; this duty applies only to the self-build path);
- `BeaconBlockRoot` matches the block root decided by the [§4](#4-modified-proposer-duty) block QBFT for the slot.

No envelope-content validation: `PayloadRoot` (and therefore every constituent field of the underlying `ExecutionPayload` such as `transactions`, `withdrawals`, `state_root`, `block_hash`) is leader-decided and trusted by the cluster. This matches the existing blinded-block trust model in `BNRoleProposer`, where operators do no field-level validation against their local BN view (see Security Considerations).

#### Post-consensus

Operators sign the decided `BlindedExecutionPayloadEnvelope`'s signing root under `DOMAIN_BEACON_BUILDER` (`0x0B000000`), domain epoch = `compute_epoch_at_slot(duty.slot)` (matching `verify_execution_payload_envelope_signature`'s `get_domain(state, DOMAIN_BEACON_BUILDER)` against the proposal-slot state); by SSZ root-equivalence this is the full envelope's signing root. Partial sigs broadcast as `PostConsensusPartialSig`.

#### Publication

Each operator's BN built a different envelope; only the operator whose blinded form matched the QBFT decision can publish. That operator reconstructs the signature and POSTs to `/eth/v1/beacon/execution_payload_envelopes` (merged via [beacon-APIs #580](https://github.com/ethereum/beacon-APIs/pull/580)); the body variant is selected by the required `Eth-Execution-Payload-Blinded` header. For stateless self-build the envelope arrived inline in `BlockContents`, so that operator holds the matching full payload, blobs, and KZG proofs and publishes `SignedExecutionPayloadEnvelopeContents` (header `false`). For stateful self-build the operator publishes `SignedBlindedExecutionPayloadEnvelope` (header `true`) to the same BN that produced the [§4](#4-modified-proposer-duty) block: the decided blinded envelope plus the reconstructed signature, from which that BN reconstructs the full envelope out of its production-time cache (any other BN lacks the cache and rejects the publish with a 400). Other operators complete without publishing.

### 7. SSV Message Validation

SSV pubsub message validation is content-agnostic: it never decodes the signed beacon object, so it enforces only metadata. This section pins those rules for the three new roles and the deprecated one; all structural, signature, and topic rules for existing roles apply unchanged. Classifications follow the existing convention: REJECT penalizes the sending peer while IGNORE drops without forwarding.

All three new roles are validator-scoped, like `RoleValidatorRegistration` and `RoleVoluntaryExit`: the `MessageID` carries the validator public key, routing reuses the cluster's existing subnet (no new topic), and each `PartialSignatureMessages` container carries at most one entry (the non-committee packet rule, REJECT above 1). Partial-signature type must match the role (REJECT on mismatch), and consensus messages are REJECT'd for the two non-QBFT roles, joining the existing registration/exit rule.

| Role (wire) | Consensus messages | Partial-sig type; count per (signer, slot) | Message slot | Earliness allowance | Lateness TTL | Duty assignment check | Duty limit per epoch |
|---|---|---|---|---|---|---|---|
| `RolePTCAttester` (7) | REJECT | `PTCAttesterPartialSig` (7); 1 | PTC attestation slot | none | 3 slots | PTC assignment at slot (IGNORE) | 2 (IGNORE) |
| `RoleProposerPreferences` (8) | REJECT | `ProposerPreferencesPartialSig` (8); up to 4 distinct signing roots, repeat of a seen root = duplicate | `proposal_slot` | `(1 + MIN_SEED_LOOKAHEAD) * SLOTS_PER_EPOCH` slots | 2 slots | proposer assignment at `proposal_slot` (IGNORE) | `SLOTS_PER_EPOCH` (IGNORE) |
| `RoleEnvelopeProposer` (9) | QBFT: 1 proposal / prepare / commit / round-change per (signer, slot, round); round cut-off 2 | `PostConsensusPartialSig` (0); 1 | proposal slot | none | 3 slots | proposer assignment at slot (IGNORE) | `SLOTS_PER_EPOCH` (IGNORE) |

Lateness TTL uses the existing deadline convention: a message is late once received after `slot_start(slot + TTL)` plus the clock-error margin. Duty limits count distinct duty slots per (signer, epoch of the message slot). Duty assignment checks apply only once the local node knows the duties for the message slot's epoch (a not-yet-fetched epoch is tolerated rather than IGNORE'd, since the message can legitimately arrive first).

A view that predates the node's most recent validator-set change is treated the same way: skip the check, don't IGNORE, until it refreshes. This is load-bearing for the one-shot `RoleProposerPreferences` (8) and `RoleEnvelopeProposer` (9) checks: their (re-)emission is triggered by the very validator-index change that invalidates a peer's cached view, and a deterministic partial dropped once is unrecoverable (gossip's seen-cache suppresses any re-broadcast), so a wrongly-dropped first copy permanently starves that (`proposal_slot`, `validator_index`) from quorum. Anti-spam stays bounded by committee membership, the distinct-signing-root cap, and the per-epoch duty-count cap.

Derivations, per row:

- **PTC (7).** One partial-signature round, no pre/post split: `PTCAttesterPartialSig` is the only type, once per duty. The vote is consumed within its own slot and included at slot + 1 ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)), so the short non-committee TTL applies. The duty limit follows from `compute_ptc` drawing members exclusively from `get_beacon_committee(state, slot, i)`: a validator belongs to exactly one slot's beacon committees per epoch, so the honest bound is one PTC duty per validator per epoch, plus the +1 reorg margin the existing registration and aggregation limits already use. Multiple PTC seats within one slot are still one duty: the validator signs one `PayloadAttestationData`, and seat multiplicity lives in the aggregation bits.
- **ProposerPreferences (8).**
  - *Earliness:* the message slot is `proposal_slot` ([§5](#5-proposer-preferences-duty)), up to the proposer lookahead in the future at emission, hence the allowance.
  - *Lateness:* the tight TTL drops stale preferences as replays.
  - *Slot-advance exempt:* a signer holds its whole lookahead at once, so a lower-slot message is a concurrent duty, not a stale one (lateness is its past bound).
  - *Dedup:* the base one-per-(signer, slot) rule gains a bounded exception for distinct signing roots at one proposal slot; a repeat of a seen root is a duplicate (same-peer REJECT, relayed IGNORE, the existing two-tier handling), and a distinct root past the cap is IGNORE'd (rate-limiting, not a provable violation). Those extra roots come from preference-input changes between emissions, notably a `dependent_root` shift under reorg ([§5](#5-proposer-preferences-duty)); the cap is policy headroom, and the epoch-boundary batch never touches it since each proposal slot is its own key.
  - *Duty limit:* at most one proposal per slot.
- **EnvelopeProposer (9).** Standard QBFT validation applies (leader round-robin, justification rules, one message per type per round); the round cut-off is the proposer-class value, and the [§6](#6-new-duty-envelope-signing-self-build-path) timing makes anything higher moot: the duty's window ends at the `PAYLOAD_DUE_BPS` cutoff (~a quarter slot, [§6](#6-new-duty-envelope-signing-self-build-path)), which fits the first round plus at most one round-change. Post-consensus is the only partial-signature phase (no pre-consensus analog to RANDAO). The duty exists only for the cluster's own proposal slots, so the proposer-assignment check applies, and the duty limit mirrors the preferences bound.

Epoch-aware fork gates, a new rule class. Both are REJECT because they are slot-scoped rather than wall-clock-scoped: the gated condition is carried in the message itself, so no honest ordering race can produce it.

| Rule | Classification | Explanation |
|---|---|---|
| Role in {7, 8, 9} and `epoch(msg.slot) < GLOAS_FORK_EPOCH` | REJECT | These duties do not exist before the fork. [§5](#5-proposer-preferences-duty)'s required pre-fork emission carries post-fork `proposal_slot` values by construction, so it passes this gate with no special case. |
| `RoleValidatorRegistration` (4) and `epoch(msg.slot) >= GLOAS_FORK_EPOCH` | REJECT | [§5](#5-proposer-preferences-duty) deprecates the duty at the fork; honest registrations are always stamped with pre-fork slots. Pre-Gloas slots remain valid and unchanged. |

## Security Considerations

### `GloasBeaconVoteValueCheckF` must include `AttestationDataIndex` in slashability checks

Under Gloas, `AttestationData.Index` is part of the attestation data root and therefore part of the double-vote slashing predicate. `GloasBeaconVoteValueCheckF` must reconstruct the full Gloas `AttestationData` with `Index` from the decided `GloasBeaconVote.AttestationDataIndex` before calling `IsAttestationSlashable`; otherwise an operator could sign `index=0` and `index=1` for the same `(source, target)` in the same slot without the predicate tripping.

### Gloas `AttestationData.Index` is trusted from the QBFT leader

The Gloas `AttestationData.Index` value check ([§2](#2-modified-attestation-duty)) does not require the QBFT-decided value to match each operator's local BN view. Requiring local agreement would fail QBFT rounds whenever operators observe fork-choice state at slightly different times around the deadline, a normal gossip-lag scenario. Accepted tradeoff: a malicious QBFT leader can push a value contrary to the cluster's majority BN observation. This matches existing ssv-spec treatment of `BeaconVote.BlockRoot`, which is trusted from the leader because BNs legitimately diverge on fork-choice head.

### PTC reconstruction is honest-convergence, not consensus

PTC runs no QBFT ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)): each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% broadcast mark, and a per-validator signature reconstructs only when a threshold of operators converged on byte-identical data. There is no leader to push a value contrary to the cluster's observation, and an operator can only ever vote its own honest observation. The cost is liveness rather than safety: when operators' beacon nodes split across observations (envelope-arrival jitter at the 50% `PAYLOAD_DUE_BPS` boundary, head or blob-availability jitter at evaluation time around `MAXIMUM_GOSSIP_CLOCK_DISPARITY`, or operators on diverged forks), no observation may reach threshold and the cluster's vote for that validator is a silent miss. That miss is non-slashable and its only effect is the foregone contribution to the `PTC_SIZE/2` fork-choice tally, bounded by SSV's PTC seat share. The off-slot-root case that Gloas gossip guards with an IGNORE-level `block.slot == data.slot` check ([`p2p-interface.md`](https://github.com/ethereum/consensus-specs/blob/801a38e1524a4945e30105a281ae693e3355d5ad/specs/gloas/p2p-interface.md#L400-L401)) cannot arise here: each operator signs the block it observed for `duty.slot`.

### Config divergence silently disables trustless builder bids

`ProposerPreferences` reconstruction requires a quorum of operators to derive the same signing root, which depends on `target_gas_limit` (per-operator config) and `dependent_root` (per-operator BN observation). Divergence on either splits signing roots; if no root reaches threshold, there is no reconstructed signature, no gossip publication, and therefore no matching preference on the `execution_payload_bid` topic; bids for the slot are IGNORE'd by gossip ([§5](#5-proposer-preferences-duty)), leaving the BN with no trustless builder options to return. Same reconstruction failure shape as `ValidatorRegistration` today.

### Too-early `SignedProposerPreferences` publication pins the wrong preference

Because the `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple ([§5](#5-proposer-preferences-duty)), reconstructing and publishing a preference before all tuple inputs are final can durably pin a wrong-input preference: a later corrected message for the same tuple is dropped by gossip rather than treated as a replacement. Builders keep using the stale preference, and bids matching the corrected values fail the [§5](#5-proposer-preferences-duty) handshake. Operators must therefore hold publication until `dependent_root`, `fee_recipient`, and `target_gas_limit` are all final for the tuple, and re-emit only when the tuple itself changes (notably when `dependent_root` shifts due to reorg, or `proposer_lookahead` reassigns the validator to a different slot). Distinct from the config-divergence entry above: there, divergence prevents publication; here, premature publication pins the wrong preference more durably than no publication would.

### Late `dependent_root` change near the proposal slot may leave the slot with no matching bid

A late `dependent_root` change tightens the re-emission window but does not block it: the changed root forms a new `(dependent_root, proposal_slot, validator_index)` tuple, which gossip accepts (first-message pinning binds a fixed tuple), so the risk here is the remaining time budget, not gossip rejection. Under non-finality, a deep reorg affecting the end-of-p-2 dependent block forces the proposer to re-emit `SignedProposerPreferences` with the new root; if the re-emission + builder-bid gossip round-trip cannot complete before the proposal deadline, the slot falls through to [§6](#6-new-duty-envelope-signing-self-build-path) self-build with a compressed envelope-signing window.

### Matching-envelope operator failure or late publication misses the slot's envelope

The slot's envelope is missed if the operator whose envelope blinds to the [§6](#6-new-duty-envelope-signing-self-build-path)-decided value (the only one holding the matching full bytes; see [§6](#6-new-duty-envelope-signing-self-build-path) Publication) fails before publishing it, or if the envelope-QBFT round completes after `get_payload_due_ms()` so the signed envelope reaches PTC validators too late to observe before the cutoff. In either case PTC records `payload_present = FALSE` ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)); proposer forfeits the payload reward. No worse than the no-envelope-signing baseline for the self-build path.

## Open Questions / Upstream Watchlist

This section is intentionally limited to upstream items that could still change the normative SSV behavior described above. If any of these settle differently, this SIP should be updated.

- beacon-APIs release status: `produceBlockV4` (`GET /eth/v4/validator/blocks/{slot}`), the [§6](#6-new-duty-envelope-signing-self-build-path) envelope publication and fetch endpoints, and the [§5](#5-proposer-preferences-duty) `SignedProposerPreferences` submission endpoint are merged to `beacon-APIs/master` ([#580](https://github.com/ethereum/beacon-APIs/pull/580), [#608](https://github.com/ethereum/beacon-APIs/pull/608)) but unreleased: the latest release tag (`v5.0.0-alpha.2`) predates them, and no client implementations are marked in the compatibility matrix yet. Watch for pre-release changes to the response variant discriminators, request/response bodies, and SSZ encodings.
- EIP-7688 progressive containers: [EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) (forward-compatible consensus data structures) reshapes core Gloas containers into `ProgressiveContainer`s, changing how `hash_tree_root` is computed for the [§4](#4-modified-proposer-duty) block signing root and the [§6](#6-new-duty-envelope-signing-self-build-path) envelope. It is only Considered for Inclusion (CFI), not Scheduled for Inclusion, for Glamsterdam ([EIP-7773](https://eips.ethereum.org/EIPS/eip-7773); targeted for glamsterdam-devnet-7, inclusion decision pending). The pinned snapshot `801a38e15` has already merged it ([consensus-specs #4630](https://github.com/ethereum/consensus-specs/pull/4630)), but this SIP describes positional merkleization until 7688 is SFI'd: at that point [§6](#6-new-duty-envelope-signing-self-build-path)'s blinded-envelope note adopts `mix_in_active_fields(merkleize_progressive(...))` and [§4](#4-modified-proposer-duty) gains a matching block-root note.
- Builder-API authenticated connections (`SignedRequestAuthV1`): builder-specs [#138](https://github.com/ethereum/builder-specs/pull/138) (merged 2026-06-11, not yet in a release) defines an optional validator-key BLS signature over `{builder URL, slot}` under the fork-agnostic domain `DOMAIN_REQUEST_AUTH` (`0x0B000001`), used to authenticate direct bid requests (`getExecutionPayloadBid`) and required for per-builder execution payments (`submitBuilderPreferences`). A bid pays the proposer via `bid.value`, deducted on-chain from the builder's staked collateral, and optionally via `bid.execution_payment`, a trusted execution-layer payment that the proposer must opt into per builder by submitting a `max_execution_payment` cap through `submitBuilderPreferences`. Deferring the duty costs no liveness in the interim: a cluster that has not yet submitted builder preferences has a zero cap, so builders MUST NOT attach execution payments and every bid it can receive pays from staked collateral only (gossiped bids MUST carry `execution_payment = 0` regardless of channel). Distributed signing of `RequestAuthV1` follows the `ValidatorRegistration` shape (validator-scoped, one partial-signature round, no QBFT) and will be specified once the upstream surface settles, meaning a standardized hop between the key holder and block assembly (a beacon-APIs successor to the now-deprecated `register_validator` forwarding, or a sidecar specification) and a builder-specs release that includes the Gloas builder API.

## Acknowledgements

Thanks to @diegomrsantos and @iurii-ssv for review, design discussion, and feedback on the Gloas integration, including PTC handling, proposer preferences, and self build envelope signing.
