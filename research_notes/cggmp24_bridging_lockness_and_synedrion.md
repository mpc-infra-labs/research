# Bridging Lockness and Synedrion for Key Refresh

## 0. Scope

This note documents the motivation, design, experiment, and limits of the bridge prototype.

The central question is:

> Can Synedrion be used for key refresh while Lockness remains the operational system used before and after refresh?

The note covers:

- why this bridge is useful
- how Lockness and Synedrion differ in threshold support, refresh support, and HD-wallet semantics
- why the bridge reduces to conversion between Shamir-style and additive share semantics
- why local conversion is available for `n-of-n`
- why `t-of-n` with `t < n` requires interaction
- what the current prototype implements
- what the result does and does not imply for production

The note is not a full survey of threshold ECDSA or CGGMP-family protocols.

---

## 1. Problem Statement

The project starts from an operational mismatch between two systems.

Lockness is used here as the operational anchor. Two public facts matter:

- the Lockness CGGMP implementation is documented as audited [R1]
- the Lockness refresh API documentation states that refresh does not work for general-threshold key shares [R2]

Synedrion is relevant for the opposite reason:

- its README lists refresh protocols as implemented [R3]
- the same README states that the library is unaudited, work in progress, and should be used at the user's own risk [R3]

The project therefore does **not** ask whether Synedrion should replace Lockness. The narrower question is whether Synedrion can be used only for the refresh step, with Lockness-compatible state on both sides of that step.

That is the bridge problem addressed here.

---

## 2. Why HD-Wallet Semantics Matter

This project treats HD-wallet semantics as a custody requirement rather than an optional feature.

Public Lockness documentation states that its HD-wallet support is based on SLIP10 and is compatible with BIP32-style workflows [R1][R4]. The public `hd-wallet` repository also documents derivation from seed to master extended key, then along a derivation path [R4]. This matches the SLIP-0010 model in which:

- the master secret and master chain code come from the seed
- non-hardened child derivation uses the parent chain code, parent public key, and child index [R5]

In custody systems, this matters because child-wallet derivation is used for:

- address fan-out under one root
- account separation
- operational wallet organization
- interoperability with HD-wallet tooling and recovery procedures

Synedrion does contain explicit BIP32-related threshold-share code paths [R14][R15]. The issue considered here is narrower. Its derivation model is not treated here as a drop-in replacement for the seed-rooted custody semantics expected from the Lockness side. One reason is that the Synedrion BIP32 code itself contains a TODO asking whether deriving the initial chain code from public information is acceptable [R16].

That point matters for evaluation. A system can support signing and refresh while still failing to match the HD-wallet assumptions of the surrounding custody stack.

---

## 3. Capability Comparison

The bridge experiment is only well motivated if the two systems have complementary properties.

| Topic | Lockness side | Synedrion side |
|---|---|---|
| Audit posture | Publicly documented as audited [R1] | Publicly documented as unaudited and work in progress [R3] |
| Refresh | Public docs say refresh does not work for general-threshold key shares [R2] | Refresh is implemented [R3] |
| Threshold support | Public docs state arbitrary threshold support was added to the library [R1] | Threshold support exists, but is exposed through `ThresholdKeyShare` and `KeyResharing`; Synedrion also states that threshold resharing is not part of CGGMP'24 proper [R3][R6][R7][R8] |
| HD-wallet semantics | Publicly positioned as SLIP10-based and BIP32-compatible [R1][R4] | Has BIP32-related threshold-share support, but is not treated here as a substitute for Lockness-side custody derivation semantics; the BIP32 code path also notes a TODO about deriving the initial chain code from public information [R14][R15][R16] |
| Role in this note | Operational anchor | Refresh engine |

Two limits should be stated explicitly.

First, the Lockness side is not presented here as solving every threshold problem. Its own refresh documentation states a limitation for general-threshold key shares [R2].

Second, the Synedrion side is not presented here as lacking threshold support. Threshold support is present, but it is exposed through an additional threshold-specific layer rather than only through the ordinary `KeyShare` path [R6][R7][R8].

---

## 4. What Threshold Support Means in Synedrion

Synedrion distinguishes between `KeyShare` and `ThresholdKeyShare` [R6].

A regular `KeyShare` is used directly in ordinary signing and refresh. A `ThresholdKeyShare` carries threshold-holder-set metadata such as:

- share IDs
- public shares
- threshold metadata
- transformations such as `to_key_share()` and `from_key_share()` [R6]

The threshold test in Synedrion makes the intended flow explicit [R7]:

1. an active signer set runs key generation
2. the resulting `KeyShare`s are converted into `ThresholdKeyShare`
3. `KeyResharing` redistributes to a wider threshold holder set
4. a selected subset later converts back to regular `KeyShare` form and signs

This matters for two reasons.

First, Synedrion threshold support is real.

Second, Synedrion itself documents threshold resharing as outside CGGMP'24 proper, even though the library includes it because threshold functionality needs it in practice [R3][R8].

This helps explain the design choice used here. Synedrion is used where its implemented refresh machinery is most useful, but the design does not assume that every threshold-state transition should be delegated to Synedrion as the long-term operational system.

---

## 5. Bridge Objective

The bridge is not only a serialization problem.

The bridge must preserve:

- the same underlying signing authority
- the same public key
- the same externally visible wallet identity

On Ethereum, one practical external check is the address derived from the public key.

For that reason, the experiment is not just "export and import shares". The experiment is:

1. generate a distributed key
2. sign before the bridge
3. refresh across the bridge
4. sign again
5. bridge back
6. sign again on the original operational side

That is the minimal end-to-end test of whether signing authority survives the round trip [R11].

---

## 6. Why the Bridge Reduces to Share-Semantics Conversion

The main difficulty is not the transport format. The main difficulty is the meaning of the transported share.

The implementation uses a portable share container defined in the bridge code [R12]. The container carries data such as:

- party index
- threshold metadata
- total party count
- scalar share
- public key metadata

The container is not itself a secret-sharing scheme. The same container can hold data that is interpreted as:

- a Shamir-style threshold share
- an additive-style active-signer share

The bridge problem therefore has three steps:

1. identify the share semantics at the source boundary
2. identify the share semantics expected at the destination boundary
3. convert without changing the underlying secret

This is why the bridge is presented here as a conversion problem between Shamir-style and additive-style representations [R12][R13].

### 6.1 Lockness-Compatible Side in the Current Implementation

In the current implementation, the Lockness-compatible side is best described as Shamir/VSS-shaped threshold state.

This is visible in the adapter code:

- the bridge reads threshold information from `core.vss_setup.min_signers` [R9]
- the bridge reconstructs VSS commitments and public shares when converting back into the Lockness-compatible object model [R9]

These are Shamir/VSS concepts rather than additive-share concepts. They refer to:

- a threshold value
- a polynomial commitment structure
- public shares attached to evaluation points

For that reason, the Lockness-compatible side is described here as Shamir-style threshold state.

### 6.2 Synedrion Side in the Current Implementation

In the current implementation, the Synedrion side is used in two different forms:

- `ThresholdKeyShare` for threshold-holder-set logic
- regular `KeyShare` for the signing and refresh path used by the bridge

The threshold code in Synedrion indicates that `to_key_share()` multiplies by an interpolation coefficient and `from_key_share()` divides by that coefficient [R6]. In other words, regular `KeyShare` is the active-signer form obtained after selecting a signer subset and applying the corresponding Lagrange weighting.

That is the point at which the bridge uses an additive interpretation:

- on the Lockness-to-Synedrion path, the current full-set demo converts a portable Shamir share into an additive share before constructing a Synedrion `KeyShare` [R11][R13]
- on the reverse path, the bridge converts the Synedrion-exported additive share back into a Shamir-style share before reconstructing the Lockness-compatible VSS state [R9][R10][R11][R13]

This note does **not** claim that one library is globally additive and the other is globally Shamir in every internal phase. The narrower statement is:

- the Lockness-compatible boundary used here is Shamir/VSS-shaped
- the Synedrion boundary used here for signing and refresh is the additive-style active-signer form

That is the mismatch the bridge has to cross.

### 6.3 Why Two CGGMP-Family Implementations Can Differ

Both systems are in the same CGGMP paper family, but that does not force them to expose the same software boundary objects.

The paper family specifies protocol families such as:

- key generation
- signing
- refresh
- security properties in the protocol model

It does not force every implementation to expose the same API boundary between:

- stored threshold state
- active signer state
- holder-set resharing objects
- HD-wallet derivation objects

Synedrion makes this especially clear because its own documentation states that threshold resharing is not part of CGGMP'24 proper, even though it is implemented in the library because threshold functionality needs it in practice [R3][R8].

The difference between the two implementations is therefore not evidence of contradiction. It is evidence that the implementations place their software boundaries differently.

---

## 7. Minimal Secret-Sharing Background

### 7.1 Additive Sharing

Let the secret be `s`.

In additive sharing:

```text
s = w_1 + w_2 + ... + w_n
```

Here:

- `s` is the secret
- `w_i` is the share held by party `i`

The secret is recovered by combining the additive shares.

### 7.2 Shamir Sharing

In Shamir sharing, the secret is the value at zero of a polynomial:

```text
f(x) = s + a_1 x + a_2 x^2 + ... + a_{t-1} x^{t-1}
```

Here:

- `s` is the secret
- `a_1, ..., a_{t-1}` are random coefficients
- the polynomial degree is at most `t - 1`

Party `i` receives:

```text
y_i = f(i)
```

The secret is:

```text
f(0) = s
```

Any `t` points determine a polynomial of degree at most `t - 1`, so any `t` shares can recover the secret.

---

## 8. Local Conversion in the Full-Set Case

The current prototype starts with the full-set case because local conversion is available there.

Assume the source state is additive:

```text
s = w_1 + w_2 + ... + w_n
```

Assume the target is an `n-of-n` Shamir-style representation over the fixed participant set `{1, 2, ..., n}`.

For that full set, reconstruction at zero has the form:

```text
s = λ_1 x_1 + λ_2 x_2 + ... + λ_n x_n
```

Here:

- `x_i` is the new share for party `i`
- `λ_i` is the public Lagrange coefficient for party `i` with respect to the full set

Because `λ_i` depends only on public indices, each party can define:

```text
x_i = w_i / λ_i
```

Then:

```text
λ_1 x_1 + λ_2 x_2 + ... + λ_n x_n
= λ_1 (w_1 / λ_1) + ... + λ_n (w_n / λ_n)
= w_1 + ... + w_n
= s
```

This gives three immediate consequences:

- party `i` needs only its own `w_i`
- party `i` needs only the public coefficient `λ_i`
- party `i` does not need any other party's secret share

That is why the full-set bridge can be local, and why the current prototype begins with `n-of-n` [R11][R13].

---

## 9. Why Threshold Conversion Requires Interaction

The case `t-of-n` with `t < n` is different.

The claim here is not that threshold bridge is impossible. The claim is that threshold bridge is not local.

Assume again:

```text
s = w_1 + w_2 + ... + w_n
```

Suppose we try to convert locally:

```text
x_i = g_i(w_i)
```

This means each new share depends only on its corresponding old share.

Now choose a threshold subset `T` of size `t`. If the new shares inside `T` can recover the secret, then the recovered value depends only on the set `{w_i : i in T}`.

But the actual secret is:

```text
s = sum over all i of w_i
```

If `t < n`, then there exists some party `k` not in `T`. Changing `w_k` changes `s`, but it does not change any `x_i` with `i in T`, because each `x_i` depends only on its own local input.

This is a contradiction.

Therefore, when `t < n`, threshold bridge cannot be purely local. Information from parties outside the final signing subset must enter the new shares. That is the reason interaction is required.

---

## 10. What Must Be Communicated in the Threshold Case

In the threshold case, each party must distribute its contribution across the future holder set.

Party `i` uses its additive share `w_i` as the constant term of a fresh random polynomial:

```text
g_i(x) = w_i + b_{i,1} x + b_{i,2} x^2 + ... + b_{i,t-1} x^{t-1}
```

Then:

```text
g_i(0) = w_i
```

Party `i` does **not** send the whole polynomial, because that would reveal `w_i` by evaluation at zero.

Instead, party `i` sends only:

```text
g_i(j)
```

to party `j`.

Party `j` receives:

```text
g_1(j), g_2(j), ..., g_n(j)
```

and forms:

```text
x_j = g_1(j) + g_2(j) + ... + g_n(j)
```

Define the global polynomial:

```text
G(x) = g_1(x) + g_2(x) + ... + g_n(x)
```

Then:

```text
x_j = G(j)
```

and:

```text
G(0) = g_1(0) + ... + g_n(0)
     = w_1 + ... + w_n
     = s
```

The resulting shares are therefore Shamir shares on a single polynomial `G(x)` whose secret is still `s`.

This is the resharing step that the current prototype does not implement at the bridge boundary, even though the code keeps a generic interactive `additive_portable_to_shamir_portable(...)` helper for that direction [R11][R13].

---

## 11. Why the Prototype Starts with `n-of-n`

The prototype is organized around the local case because that isolates the bridge question from the resharing question.

The question addressed by the current prototype is:

> Can the project build a useful refresh bridge without adding another interactive resharing protocol at the bridge boundary?

The cleanest test case for that question is `n-of-n`.

In the full-set case:

- the bridge conversion stays local
- the bridge does not add a new communication protocol
- the experiment isolates whether signing authority survives the round trip

This does **not** imply that threshold bridge is impossible. It only means:

- `n-of-n` is the local case
- `t-of-n` with `t < n` is the interactive case

The current implementation covers the local case [R11].

---

## 12. What the Prototype Implements

The Lockness-compatible side is represented in the prototype implementation by the CGGMP24-compatible bridge path. Synedrion is used for refresh [R9][R10][R11].

The implemented end-to-end flow is:

1. generate a distributed key on the Lockness-compatible side
2. sign a transaction before the bridge
3. export to portable form
4. convert into Synedrion-compatible state
5. run Synedrion refresh
6. sign again on the Synedrion side
7. export again
8. convert back into Lockness-compatible state
9. sign again on the Lockness-compatible side

The current implementation runs this in the full-set case so that the bridge conversion itself remains local [R11].

This is the implemented result of the current prototype.

---

## 13. Integration Strategies Considered

Development considered two implementation strategies.

### 13.1 Strategy A: Fork or Clone the Upstream Library

Strategy A is:

- clone the upstream library
- expose private constructors or fields
- add bridge-specific helper methods inside the upstream code

The main benefit is tighter access to internal types.

The main costs are:

- dependence on a modified fork
- ongoing merge and maintenance burden
- weaker interoperability claims, because the result depends on patched upstream code

For those reasons, the implementation did not use the fork-first path as the main strategy.

### 13.2 Strategy B: JSON / Serde Adapter Layer

The implemented bridge uses a JSON-based adapter layer [R9][R10][R11].

The pattern is:

- serialize upstream objects into JSON-like values
- extract or patch the required fields
- deserialize into the destination object model

This appears in the bridge code on both sides [R9][R10][R11]. Examples include:

- extracting Lockness-compatible secret-share and threshold metadata from serialized fields
- constructing Synedrion `KeyShare` through JSON because the exact public constructor needed by the bridge is not exposed
- patching `public` share lists and ID types through JSON round-trips

This avoids maintaining a fork, but it introduces its own risks.

### 13.3 Problems Introduced by the JSON Strategy

The JSON strategy creates at least five classes of risk.

#### Schema fragility

The bridge depends on field names and serialized layout. Upstream serialization changes can break the bridge even if the cryptographic meaning of the library has not changed.

#### Invariant bypass

Rebuilding objects from JSON bypasses private constructors and internal invariants. An object may deserialize successfully while still violating intended library invariants.

#### Semantic ambiguity

The same portable container can hold either:

- a Shamir-style share
- an additive-style share

JSON does not enforce that distinction. The meaning depends on surrounding context. This was a recurrent source of confusion during development.

#### Manual patching burden

The project must manually patch:

- public-share lists
- ID types
- commitment-related metadata when rebuilding the Lockness-compatible side

This is less robust than a typed public bridge API.

#### Debugging overhead

Several hard failures during development were adapter failures rather than cryptographic failures, including:

- type mismatch between `u16` and verifier wrapper types
- incomplete public-share lists
- stale metadata after transitions between additive-style and Shamir-style interpretations

### 13.4 Why JSON Was Still Used

The JSON strategy still has one main advantage here:

- it tests bridge feasibility against the libraries as they currently exist

This makes the result closer to an interoperability test than a result that depends on a patched upstream fork.

---

## 14. What the Prototype Does Not Implement

The current implementation does **not** implement a full threshold bridge protocol at the bridge boundary.

In particular, it does not add a bridge-stage protocol for:

- threshold resharing
- VSS-style consistency checks
- holder-set transition logic across the bridge boundary

The current result must therefore be read with care.

### 14.1 Implemented and Demonstrated

The current implementation covers the following end-to-end flow:

- Lockness-compatible key generation
- signing before the bridge
- conversion into Synedrion-compatible state
- refresh inside Synedrion
- signing after refresh
- conversion back into Lockness-compatible state
- signing again after the round trip

These steps correspond to the main bridge flow and supporting conversion helpers in the prototype implementation [R9][R10][R11][R13].

### 14.2 Reported Outcome of the Current Experiment

The project run summarized by this note reported success for the full-set flow listed above:

- key generation on the Lockness-compatible side
- signing before the bridge
- refresh on the Synedrion side
- signing after refresh
- conversion back into Lockness-compatible state
- signing again after the round trip

This reported outcome is narrower than a production claim. It is a statement about the current full-set experiment.

### 14.3 Theoretical but Not Implemented Here

The following claim is theoretical rather than implemented in the current prototype:

- a threshold bridge with `t < n` should be possible in principle

Making that claim operational would require the bridge layer itself to become interactive.

---

## 15. Production Implications

The implementation covers, and the current experiment reports, a local bridge in the full-set case. This is not the same as a production-ready threshold bridge.

The main production implications are the following.

### 15.1 Trusted Surface Expansion

If the bridge process sees enough share material to perform both directions of conversion, the trusted computing base becomes larger.

Even if the code never explicitly reconstructs the full private key in a single variable, a central conversion service may still occupy a role much closer to key reconstruction than a normal MPC node should.

### 15.2 Operational Cost of `n-of-n`

The full-set case avoids bridge-side resharing, but it also removes threshold liveness properties:

- all parties must be present
- liveness is worse
- fault tolerance disappears

`n-of-n` is therefore simpler for the bridge, but not automatically better for the product.

### 15.3 HD-Wallet Semantics Remain Relevant

Even if the bridge preserves signing ability, signing ability is not the only production invariant.

Custody systems also care about:

- child-wallet derivation semantics
- backup and recovery assumptions
- audit boundaries
- wallet-identity semantics under existing tooling

For that reason, the BIP32 / SLIP10 discussion is part of the evaluation rather than an unrelated side topic.

### 15.4 Threshold Bridge Remains a Valid Direction

The project should not be read as a claim against threshold bridge.

The current result is narrower:

- the local full-set case is implementable
- the threshold case requires a larger bridge protocol

That is the conclusion supported by the current implementation.

---

## 16. Conclusion

The two systems address different parts of the operational problem.

Lockness is used as the operational anchor in this design, but it does not provide the refresh path needed here [R1][R2]. Synedrion provides refresh, but the public documentation gives it a weaker production trust posture, and its threshold path relies on resharing machinery outside CGGMP'24 proper [R3][R6][R7][R8].

Those differences motivate the narrower bridge experiment described in this note.

In the reported full-set experiment, a Lockness-originated signing authority:

- moved into Synedrion for refresh
- moved back into Lockness-compatible state
- continued to sign in a full-set bridge configuration

This statement refers to the full-set bridge flow implemented in the prototype implementation [R11][R13].

The next step, if needed, is not a new mathematical primitive. It is implementation of the bridge itself as a threshold resharing participant [R8][R11].

---

## References

- [R1] Lockness CGGMP docs: arbitrary threshold support, audited status, HD-wallet support.  
  <https://lfdt-lockness.github.io/cggmp21/src/cggmp21/lib.rs.html>

- [R2] Lockness key refresh docs: refresh does not work with general-threshold key shares.  
  <https://lfdt-lockness.github.io/cggmp21/cggmp21/fn.key_refresh.html>

- [R3] Synedrion README: unaudited status, implemented protocols, threshold resharing not part of CGGMP'24 proper.  
  <https://github.com/entropyxyz/synedrion>

- [R4] Lockness `hd-wallet` README: SLIP10-based HD-wallet derivation and extended-key examples.  
  <https://github.com/LFDT-Lockness/hd-wallet>

- [R5] SLIP-0010 specification: seed-derived master key and chain code, child derivation rules.  
  <https://github.com/satoshilabs/slips/blob/master/slip-0010.md>

- [R6] Synedrion threshold-share implementation.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/entities/threshold.rs>

- [R7] Synedrion threshold test flow.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/tests/threshold.rs>

- [R8] Synedrion key resharing protocol.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/protocols/key_resharing.rs>

- [R9] Lockness bridge adapter used by the prototype. Local code inspection: `src/bridge/cggmp.rs`.

- [R10] Synedrion bridge adapter used by the prototype. Local code inspection: `src/bridge/synedrion.rs`.

- [R11] Main bridge flow used by the prototype. Local code inspection: `src/main.rs`.

- [R12] Portable share type used by the prototype. Local code inspection: `src/bridge/common.rs`.

- [R13] Share-conversion logic used by the prototype. Local code inspection: `src/bridge/core.rs`.

- [R14] Synedrion public library docs: `bip32` feature enables BIP32 support for `ThresholdKeyShare`.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/lib.rs>

- [R15] Synedrion threshold-share BIP32 derivation code.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/entities/threshold.rs>

- [R16] Synedrion BIP32 helper code, including TODO about deriving the initial chain code from public information.  
  <https://github.com/entropyxyz/synedrion/blob/master/src/curve/bip32.rs>
