# ECC, Signature Schemes, and MPC

## 0. Scope

This note covers:

- minimal elliptic-curve notation required for signature schemes
- ECDSA, Schnorr, and EdDSA at the equation level
- threshold-signing implications of each scheme
- the source of complexity in threshold ECDSA
- the nonce-coordination issue in threshold EdDSA and FROST-style systems

This note does not attempt to provide:

- a full introduction to elliptic-curve arithmetic
- full proofs of security
- a full survey of threshold-signature literature
- implementation details for every deployed library

The focus is restricted to the aspects of these schemes that matter for MPC and threshold signing.


## 1. Motivation

In blockchain systems, transaction authorization reduces to signature verification. A valid signature demonstrates control of the signing key associated with the relevant address or account.

Institutional custody systems cannot usually tolerate a model in which one machine or one operator can produce a signature unilaterally. Threshold signing addresses this by distributing signing capability across multiple parties while preserving the external interface of a standard single-key wallet. The chain still sees one public key and one valid signature. The trust distribution exists entirely off-chain.

The underlying signature scheme determines how difficult this is. Threshold signing is not equally easy across ECDSA, Schnorr, and EdDSA. The main difference comes from the algebra of the signing equation.


## 2. Minimal ECC Notation

Let:

- $G$ be a generator point of a cyclic elliptic-curve group
- $n$ be the order of $G$
- lowercase variables denote scalars modulo $n$
- uppercase variables denote curve points

Basic operations:

- scalar multiplication: $a \cdot G$
- point addition: $P + Q$

Key derivation:

- private key: $d$
- public key: $Q = d \cdot G$

The discrete-log assumption is that recovering $d$ from $(G, Q)$ is computationally infeasible.


## 3. ECDSA

### 3.1 Notation

- $d$: private key scalar
- $Q = d \cdot G$: public key
- $z = \text{hash}(msg)$: message digest interpreted as a scalar
- $k$: per-signature nonce
- $R = k \cdot G$
- $r = x(R)$ interpreted modulo $n$
- signature: $(r, s)$

### 3.2 Signing Equation

$$s = k^{-1}(z + r d) \pmod{n}$$

### 3.3 Verification Relation

Starting from:

$$s = k^{-1}(z + r d)$$

multiply by $k$:

$$k s = z + r d$$

solve for $k$:

$$k = \frac{z + r d}{s}$$

multiply by $G$:

$$k \cdot G = \frac{z}{s} \cdot G + \frac{r}{s} \cdot (d \cdot G)$$

substitute $Q = d \cdot G$:

$$k \cdot G = \frac{z}{s} \cdot G + \frac{r}{s} \cdot Q$$

The verifier reconstructs:

$$R' = \frac{z}{s} \cdot G + \frac{r}{s} \cdot Q$$

The signature is accepted if:

$$x(R') = r$$

### 3.4 Structural Property Relevant to MPC

The signing equation contains a modular inverse of a secret nonce:

$$k^{-1}$$

This introduces non-linearity into threshold signing.

### 3.5 Visualization

[Interactive demo](https://www.desmos.com/calculator/pxrkkhxcez)

<img src="./assets/ecdsa_demo.gif" alt="ECDSA interactive demo" width="640" />

The visualization is not a literal secp256k1 plot. It is a visual proxy for the algebraic relation between the nonce point, the reconstructed verification point, and the signature equation.


## 4. Schnorr

### 4.1 Notation

- $a$: private key scalar
- $A = a \cdot G$: public key
- $r$: nonce scalar
- $R = r \cdot G$
- $c = \text{hash}(R, A, msg)$: challenge scalar
- signature: $(R, S)$

### 4.2 Signing Equation

$$S = r + c a$$

### 4.3 Verification Relation

Multiply by $G$:

$$S \cdot G = r \cdot G + c \cdot (a \cdot G)$$

Substitute:

$$S \cdot G = R + c \cdot A$$

The verifier recomputes $c$ from public data and checks:

$$S \cdot G = R + c \cdot A$$

### 4.4 Structural Property Relevant to MPC

The signing equation is additive. No inversion term appears in the signing relation.


## 5. EdDSA

### 5.1 Position Relative to Schnorr

EdDSA is a concrete Schnorr-family construction. It fixes:

- the curve
- the hash function
- the encoding rules
- nonce derivation

For schematic comparison with Schnorr, the core relation may still be written as:

$$S = r + c a$$

with verification:

$$S \cdot G = R + c \cdot A$$

Actual Ed25519 signing includes additional details around hashing, clamping, encoding, and cofactor handling.

### 5.2 Deterministic Nonce Derivation

In standard single-device EdDSA, the nonce is derived deterministically from secret-derived material and the message rather than sampled directly from fresh external randomness.

This is the main property that matters in threshold settings. The core signing relation remains Schnorr-like and additive, but the standard nonce-derivation procedure does not transfer directly to a distributed setting.


## 6. Comparison

| Property | ECDSA | Schnorr | EdDSA |
|----------|-------|---------|-------|
| Core signing equation | $s = k^{-1}(z + rd)$ | $S = r + ca$ | $S = r + ca$ plus scheme-specific derivation and encoding details |
| Inversion in signing relation | Yes | No | No in the core signing relation |
| Algebraic structure | Non-linear | Additive | Additive at the signature-equation level |
| Standard nonce model | Random | Usually random | Deterministic in standard single-device form |
| Threshold-signing implications | Requires extra machinery for secret multiplication | Naturally compatible with additive sharing | Compatible with additive sharing, but nonce handling requires coordination |


## 7. Threshold ECDSA

### 7.1 Problem Statement

Threshold ECDSA requires multiple parties to jointly compute a valid ECDSA signature without reconstructing the full private key on any single machine.

The difficulty comes from the signing equation:

$$s = k^{-1}(z + r d) \pmod{n}$$

Both $d$ and $k$ are secret. In a threshold setting, they are represented in distributed form.

### 7.2 Two-Party Explanatory Form

For intuition, consider a multiplicative two-party representation:

$$x = x_1 x_2 \pmod{n}$$

and similarly for the nonce:

$$k = k_1 k_2 \pmod{n}$$

This is not the only threshold representation used in practice, but it is sufficient to expose the source of the non-linearity.

The parties compute:

$$Q_1 = x_1 \cdot G,\qquad Q_2 = x_2 \cdot G$$

and the joint public key:

$$Q = x_1 \cdot Q_2 = x_2 \cdot Q_1 = (x_1 x_2) \cdot G$$

For nonce generation:

$$R_1 = k_1 \cdot G,\qquad R_2 = k_2 \cdot G$$

and:

$$R = k_1 \cdot R_2 = k_2 \cdot R_1 = (k_1 k_2) \cdot G$$

with:

$$r = x(R)$$

### 7.3 Cross-Product Obstruction

Substitute the distributed quantities into the ECDSA equation:

$$s = (k_1 k_2)^{-1}(z + r x_1 x_2) \pmod{n}$$

If $\gamma_i = k_i^{-1}$, then:

$$s = \gamma_1 \gamma_2 z + r \cdot \gamma_1 x_2 \cdot \gamma_2 x_1 \pmod{n}$$

The terms:

- $\gamma_1 x_2$
- $\gamma_2 x_1$

mix secrets held by different parties. These are cross-products of private values. No party can evaluate them locally without learning the other party's secret.

This is the main algebraic obstacle in threshold ECDSA.

### 7.4 Additive-Sharing View of the Same Obstacle

Under additive sharing, multiplication produces cross-terms:

$$(a_1 + a_2)(b_1 + b_2) = a_1 b_1 + a_1 b_2 + a_2 b_1 + a_2 b_2$$

The problematic terms are:

- $a_1 b_2$
- $a_2 b_1$

These cannot be computed without additional cryptographic machinery.

### 7.5 MtA

Practical threshold ECDSA protocols solve this using multiplicative-to-additive conversion, usually abbreviated MtA.

The goal of MtA is to obtain additive shares $(\alpha, \beta)$ of a product $a b$ such that:

$$\alpha + \beta = a b \pmod{n}$$

without revealing $a$ to the holder of $b$ or $b$ to the holder of $a$.

### 7.6 Paillier-Based MtA

A common approach uses Paillier additive homomorphic encryption.

Relevant properties:

$$\text{Enc}(a) \cdot \text{Enc}(b) = \text{Enc}(a + b)$$

$$\text{Enc}(a)^c = \text{Enc}(c a)$$

Sketch:

1. one party encrypts $a$
2. the other homomorphically incorporates $b$ and a random mask
3. decryption yields one additive share
4. the masking party keeps the complementary share

The result is an additive sharing of the product rather than disclosure of either multiplicand.

### 7.7 Additional Components in Practical Threshold ECDSA

Practical protocols require more than MtA alone. Typical components include:

- proofs of correct key generation
- proofs of correct nonce generation
- proofs of correct range or well-formedness
- identifiable-abort logic
- presigning phases
- key refresh or resharing

These are protocol-level protections layered around the same algebraic core problem.

### 7.8 Schnorr Proofs of Knowledge Inside Threshold ECDSA

Threshold ECDSA protocols commonly use Schnorr proofs of knowledge during setup and presigning to prove knowledge of discrete logs of contributed public points.

A standard Schnorr proof relation has the form:

$$S \cdot G = R + c \cdot Q_i$$

This is distinct from the final ECDSA signature. The point is to prove knowledge of a secret corresponding to a contributed public point without revealing that secret.


## 8. Threshold Schnorr and Threshold EdDSA

### 8.1 Additive Compatibility

In Schnorr-style schemes, additive sharing aligns with the signing equation:

$$S = r + c a$$

For additive shares:

$$a = \sum_i a_i,\qquad r = \sum_i r_i$$

the public quantities compose as:

$$A = \sum_i a_i \cdot G$$

$$R = \sum_i r_i \cdot G$$

Each party can compute a partial signature:

$$S_i = r_i + \lambda_i a_i c$$

and the aggregator computes:

$$S = \sum_i S_i$$

The verification equation remains:

$$S \cdot G = R + c \cdot A$$

The core signing relation does not require an MtA layer.

### 8.2 Threshold Generalization

Real threshold schemes usually use Shamir secret sharing rather than simple two-party additive splits. The main algebraic conclusion is unchanged: the signing equation remains compatible with linear sharing.

### 8.3 EdDSA Nonce Issue

The main complication in threshold EdDSA is not the signature equation. It is nonce derivation.

Single-device EdDSA typically uses deterministic nonce derivation:

$$r = \text{hash}(\text{prefix} \| msg)$$

In a distributed setting, no single party holds the complete secret-derived material needed to evaluate the standard nonce function directly.

As a result, threshold EdDSA systems replace single-device deterministic nonce generation with coordinated randomness plus session binding.

### 8.4 Commit-and-Reveal

One baseline construction is commit-and-reveal:

1. each party samples $r_i$
2. each party computes $R_i = r_i \cdot G$
3. each party commits to $R_i$
4. after collecting commitments, each party reveals $R_i$
5. each reveal is checked against its prior commitment
6. the aggregate nonce point is computed:

$$R = \sum_i R_i$$

This prevents adaptive last-actor nonce choice after observing others' contributions.

### 8.5 FROST

FROST improves the online signing flow by preprocessing nonce material and binding each signer's contribution to the session.

Each signer holds nonce commitments $(D_i, E_i)$ and computes:

$$\rho_i = \text{hash}\bigl(i,\ \{(D_j, E_j)\}_{j \in \text{signers}},\ msg\bigr)$$

then:

$$R_i = D_i + \rho_i E_i$$

and:

$$R = \sum_i R_i$$

The binding factor $\rho_i$ ties the nonce contribution to the signer set and message, preventing malicious recombination of preprocessed nonce material across sessions.

### 8.6 Ed25519 Key-Derivation Issue

Ed25519 key derivation includes non-linear preprocessing:

`seed -> hash -> clamp -> scalar`

For this reason, production systems often separate:

- seed-share handling for backup or recovery workflows
- scalar-share handling for signing

This is an implementation issue specific to concrete EdDSA instantiations such as Ed25519. It is separate from the main algebraic comparison between ECDSA and Schnorr-style signing.


## 9. Summary

- ECDSA signing is non-linear because the signing equation contains $k^{-1}$.
- Distributed ECDSA signing therefore requires mechanisms for secure multiplication across secrets held by different parties.
- Practical threshold ECDSA protocols use MtA and related proof systems to handle these cross-products safely.
- Schnorr signing is additive and therefore compatible with linear secret-sharing structures.
- EdDSA inherits the additive core of Schnorr-style signing.
- Threshold EdDSA is mainly complicated by nonce derivation and nonce-coordination requirements rather than by the core signature equation.


## 10. References

### 10.1 Papers

[Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)

[CGGMP21 — UC Non-Interactive, Proactive, Threshold ECDSA with Identifiable Aborts](https://eprint.iacr.org/2021/060)

[FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)

[IETF Draft — Two-Round Threshold Schnorr Signatures with FROST](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)

### 10.2 Implementations

[Taurus multi-party-sig](https://github.com/taurusgroup/multi-party-sig)

[lfdt-cggmp21](https://github.com/LFDLFoundation/cggmp21)

[synedrion](https://github.com/nucypher/synedrion)
