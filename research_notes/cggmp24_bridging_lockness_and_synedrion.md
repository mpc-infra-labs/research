# Bridging Lockness and Synedrion for Key Refresh

## 0. Scope

This note explains why this project exists, what it demonstrates, and what it does not demonstrate.

The main question is:

> Can Synedrion be used only for key refresh, while Lockness remains the main operational system?

This note focuses on:

- the practical reason to attempt this bridge
- the capability differences between Lockness and Synedrion
- the bridge problem as a secret-sharing conversion problem
- why `n-of-n` bridge can be local
- why `t-of-n` bridge needs interaction when `t < n`
- what this project implemented and tested
- what the production implications are

This note does not try to be a full survey of threshold cryptography.

---

## 1. Why This Project Exists

This project starts from an operational mismatch.

Lockness is the more trusted system in this setting because the public Lockness CGGMP implementation has been audited, and because it is closer to the production system this project wants to preserve.[R1] At the same time, public Lockness documentation also says that key refresh does not work for general-threshold key shares.[R2]

Synedrion is attractive for the opposite reason. It explicitly implements key refresh and auxiliary-info refresh.[R3] But Synedrion also states that the library is still a work in progress, has not been audited, and should be used at the user's own risk.[R3]

So the motivation is not "replace Lockness with Synedrion." The motivation is narrower:

- keep Lockness as the trusted operational anchor
- use Synedrion for the refresh capability that is missing on the Lockness side
- return to a Lockness-compatible signing state after refresh

This is the bridge problem.

---

## 2. Why BIP32 and Child-Wallet Derivation Matter

This project also treats HD-wallet semantics as a real product requirement.

Public Lockness documentation says its HD-wallet support is based on SLIP10 and is compatible with BIP32-style workflows.[R1][R4] The separate Lockness `hd-wallet` repository also documents SLIP10-based derivation from a seed to a master extended key, followed by child derivation along a path.[R4] This is aligned with the standard SLIP-0010 / BIP32 extended-key model, where:

- the master secret and master chain code come from the seed
- non-hardened child derivation uses the parent chain code, parent public key, and child index[R5]

In this project, that matters because child-wallet derivation is important in custody systems. It is used for:

- many deposit addresses under one root
- account separation without repeated key ceremonies
- operational wallet organization
- integration with existing HD-wallet tooling

For that reason, HD-wallet semantics are part of the system comparison, not an optional feature.
å
The concern on the Synedrion side is not that it has no derivation code at all. It does have BIP32-related code paths for threshold shares.[R6][R7] The concern is that, in this project, its derivation model is not trusted as a drop-in replacement for the seed-rooted custody semantics expected from the Lockness side.

So even if Synedrion can refresh keys, that does not automatically make it acceptable as a full custody replacement.

---

## 3. Capability Comparison

The bridge experiment is rational only if the two systems have complementary strengths and weaknesses.

| Topic | Lockness side | Synedrion side |
|---|---|---|
| Audit status | Publicly documented as audited.[R1] | Publicly documented as unaudited and work in progress.[R3] |
| Refresh | Public docs say refresh does not work for general-threshold key shares.[R2] | Refresh is implemented and is a main feature.[R3] |
| Threshold support | Public docs say arbitrary threshold support was added to the library.[R1] | Threshold support exists, but uses `ThresholdKeyShare` and `KeyResharing`, and Synedrion says resharing is not part of CGGMP'24 proper.[R3][R6][R7][R8] |
| HD-wallet semantics | Publicly positioned as SLIP10-based and BIP32-compatible.[R1][R4] | Has BIP32-related threshold-share support, but this project does not treat it as a full substitute for Lockness-side custody derivation semantics.[R6][R7] |
| Role in this project | Operational anchor | Refresh engine |

Two clarifications are important.

First, this table does **not** mean that Lockness solves every threshold problem out of the box. Public Lockness documentation explicitly says refresh does not work for general-threshold keys.[R2]

Second, this table does **not** mean that Synedrion lacks threshold support. It does support threshold workflows. But in Synedrion the threshold path is exposed through extra threshold-specific machinery such as `ThresholdKeyShare`, conversion between threshold and regular shares, and `KeyResharing`.[R6][R7][R8]

That distinction is important for this project.

---

## 4. What Threshold Support Means in Synedrion

It is easy to use the word "share" too loosely. Synedrion has a real distinction between:

- `KeyShare`
- `ThresholdKeyShare`

These are not the same object.[R6]

A regular `KeyShare` is the object used directly by the ordinary signing and refresh flow. A `ThresholdKeyShare` is a threshold-aware object that carries:

- share IDs
- public shares
- threshold metadata
- threshold-specific transformations such as `to_key_share()` and `from_key_share()`[R6]

The Synedrion threshold test makes the intended flow explicit.[R7] In that test:

1. an initial active signing set runs key generation
2. the resulting regular `KeyShare`s are converted into `ThresholdKeyShare`
3. `KeyResharing` distributes threshold shares to a wider holder set
4. later, a selected threshold subset converts back to regular signing shares and signs

This means Synedrion threshold support is real, but it is not just "run the normal protocol and you automatically have arbitrary threshold support for every holder configuration." There is an additional threshold layer.

Synedrion's own README is consistent with this. It lists "Threshold Key Resharing" separately and says it is not part of CGGMP'24 proper, even though it is needed to enable threshold functionality in practice.[R3]

This is one of the reasons the bridge experiment is reasonable:

- Lockness is the trusted operational side
- Synedrion has the refresh machinery
- Synedrion threshold support exists, but with more protocol surface and a weaker trust story

That makes "use Synedrion only for refresh" a reasonable thing to test.

---

## 5. The Bridge Goal

The bridge is not just a format conversion.

The bridge must preserve:

- the same underlying signing authority
- the same public key
- therefore the same externally visible wallet identity

In Ethereum terms, the strongest easy check is the address derived from the public key.

So the real bridge question is:

> Can the same signing authority start on the Lockness side, move into Synedrion for refresh, move back, and still produce valid signatures under the same public key?

That is why the experiment is "DKG, sign, refresh, bridge back, sign again" rather than just "serialize and deserialize shares."

---

## 6. Why the Bridge Reduces to Secret-Sharing Conversion

The core problem is not Rust structs. The core problem is the meaning of the shares.

This project uses a portable share container. That container carries:

- party index
- threshold metadata
- total party count
- scalar share
- public key metadata

But the container is only a transport format. It is not itself a cryptographic scheme.

The same container can represent:

- a Shamir share
- an additive share

So the bridge problem is:

1. identify what secret-sharing form the source side is using at the bridge boundary
2. identify what secret-sharing form the destination side expects
3. convert between the two without changing the underlying secret

This is why the bridge reduces to additive-vs-Shamir conversion.

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

The secret appears only when the additive shares are combined.

### 7.2 Shamir Sharing

In Shamir sharing, the secret is encoded as the value at zero of a polynomial:

```text
f(x) = s + a_1 x + a_2 x^2 + ... + a_{t-1} x^{t-1}
```

Here:

- `s` is the secret
- `a_1, ..., a_{t-1}` are random coefficients
- the degree is at most `t - 1`

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

## 8. Why `n-of-n` Bridge Can Be Local

This section explains the design choice in the current prototype.

Assume the source state is additive:

```text
s = w_1 + w_2 + ... + w_n
```

Suppose the target is an `n-of-n` Shamir-style representation over the fixed participant set `{1, 2, ..., n}`.

For that full set, the Lagrange reconstruction formula at zero is:

```text
s = λ_1 x_1 + λ_2 x_2 + ... + λ_n x_n
```

Here:

- `x_i` is the new share for party `i`
- `λ_i` is the Lagrange coefficient for party `i` with respect to the full participant set

The key fact is that `λ_i` is public. It depends only on:

- the participant indices
- the target reconstruction point `0`

It does not depend on secret data.

So each party can define its new share locally as:

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

This means:

- party `i` only needs its own `w_i`
- party `i` only needs the public coefficient `λ_i`
- party `i` does not need any other party's secret share

That is why `n-of-n` bridge can be local.

This is the reason the current prototype starts with a full-set experiment.

---

## 9. Why Threshold Bridge Requires Interaction

This is the case `t-of-n` with `t < n`.

The point here is not that threshold bridge is impossible. The point is that threshold bridge is not local.

Assume again that the source state is additive:

```text
s = w_1 + w_2 + ... + w_n
```

Now suppose we want new shares such that any subset of size `t` can recover the secret.

Assume, for contradiction, that the conversion is local:

```text
x_i = g_i(w_i)
```

That means each new share depends only on the corresponding old share.

Now pick a threshold subset `T` of size `t`. If the new shares inside `T` can recover the secret, then the recovered value depends only on `{w_i : i in T}`.

But the actual secret is:

```text
s = sum over all i of w_i
```

If `t < n`, there is some party `k` not in `T`. Changing `w_k` changes the secret, but it does not change any `x_i` inside `T`, because each `x_i` depends only on the local input `w_i`.

That is a contradiction.

So when `t < n`, a threshold bridge cannot be purely local.

Some information from parties outside the final signing subset must enter the new shares. That is exactly why communication is needed.

---

## 10. What Must Be Communicated in the Threshold Case

In the threshold case, each party must distribute its contribution across the future holder set.

Party `i` treats its additive share `w_i` as the constant term of a fresh random polynomial:

```text
g_i(x) = w_i + b_{i,1} x + b_{i,2} x^2 + ... + b_{i,t-1} x^{t-1}
```

This guarantees:

```text
g_i(0) = w_i
```

Party `i` does **not** send the whole polynomial. That would reveal `w_i` by evaluation at zero.

Instead, party `i` sends to party `j` only the point value:

```text
g_i(j)
```

Party `j` receives:

```text
g_1(j), g_2(j), ..., g_n(j)
```

and forms:

```text
x_j = g_1(j) + g_2(j) + ... + g_n(j)
```

Now define the total polynomial:

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

So the resulting shares are Shamir shares on a single global polynomial `G(x)` whose secret is still `s`.

This is the threshold resharing step that the current prototype does not implement at the bridge layer.

---

## 11. Why the Current Prototype Starts with `n-of-n`

The prototype starts with the local case because it isolates the bridge question.

The actual question is not:

> Can threshold bridge ever exist?

The actual question is:

> Can this project get a useful refresh bridge without adding another interactive resharing protocol at the bridge boundary?

If the answer is yes, the cleanest test case is `n-of-n`.

In that case:

- the bridge conversion can stay local
- the bridge does not need to introduce a new communication protocol
- the experiment can focus on whether signing authority survives the round trip

This does **not** mean threshold bridge is impossible.

It means:

- `n-of-n` bridge is the local case
- `t-of-n` bridge is the interactive case

The current prototype implements the local case.

---

## 12. What the Prototype Implements

In this project, the Lockness side is represented by the CGGMP24-compatible path in the codebase. Synedrion is used for refresh.

The implemented experiment is:

1. generate a distributed key on the Lockness-compatible side
2. sign a transaction before the bridge
3. export to the portable format
4. convert into Synedrion-compatible state
5. run Synedrion refresh
6. sign again on the Synedrion side
7. export again
8. convert back into Lockness-compatible state
9. sign again on the Lockness-compatible side

In the current implementation, this is done in the full-set case so that the bridge conversion itself remains local.

This is the current project result.

---

## 13. What the Prototype Does Not Yet Implement

The current prototype does **not** implement a full threshold bridge protocol at the bridge boundary.

In particular, it does not yet add an extra bridge-stage protocol for:

- threshold resharing
- VSS-style consistency checks
- holder-set transition logic across the bridge boundary

So the current result should be read carefully.

### 13.1 Implemented and demonstrated

The current project demonstrates:

- Lockness-compatible key generation
- signing before bridge
- conversion into Synedrion-compatible state
- key refresh inside Synedrion
- signing after refresh
- conversion back into Lockness-compatible state
- signing again after the round trip

### 13.2 Theoretical but not implemented in this prototype

The following statement is theoretical, not yet an implementation claim:

- a threshold bridge with `t < n` should also be possible in principle

But to turn that into an implemented result, the bridge layer itself must become interactive.

---

## 14. What This Means for Production

The experiment shows a feasible local bridge in the full-set case. That is a meaningful result. But it is not the same thing as a production-ready threshold bridge.

The main production implications are:

### 14.1 The bridge increases the trusted surface

If the bridge service sees enough share material to perform both directions of conversion, the trusted computing base gets larger.

Even if the code never explicitly reconstructs the full private key in one variable, a central conversion process may still occupy a role that is much closer to key reconstruction than a normal MPC node should.

### 14.2 `n-of-n` is simpler but weaker operationally

The full-set case avoids bridge-side resharing, but it has costs:

- all parties must be present
- liveness is worse
- fault tolerance disappears

So `n-of-n` is easier for the bridge, but not automatically better for the product.

### 14.3 HD-wallet semantics remain a production concern

Even if the bridge preserves signing ability, that is not the only production invariant.

Custody systems also care about:

- child-wallet derivation semantics
- backup and recovery assumptions
- audit boundaries
- wallet-identity semantics under existing tooling

This is why the BIP32 / SLIP10 discussion belongs in the evaluation.

### 14.4 Threshold bridge is still a valid future direction

This project should not be read as a proof against threshold bridge.

It should be read as:

- a proof that local bridge is possible in the full-set case
- a statement that threshold bridge requires a larger protocol implementation

That is a narrower and more accurate conclusion.

---

## 15. Bottom Line

This project is reasonable because the two systems solve different parts of the operational problem.

Lockness provides the stronger operational trust anchor in this setting, but not the refresh path needed here.[R1][R2] Synedrion provides refresh, but with a weaker production trust story and a threshold path that relies on extra resharing machinery outside CGGMP'24 proper.[R3][R6][R7][R8]

That makes a narrow bridge experiment worth doing.

The current prototype shows:

- a Lockness-originated signing authority can be moved into Synedrion for refresh
- moved back again
- and still used for signing

in a full-set bridge configuration.

The next step, if needed, is not a new mathematical idea. It is to implement the bridge itself as a threshold resharing participant.

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
