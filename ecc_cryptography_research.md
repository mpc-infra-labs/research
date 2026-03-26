# ECC Cryptography & MPC Research Notes

---

## Part 0: Motivation and Applications

Every blockchain transaction is, at its core, a math problem. You prove you own an address by producing a signature that only the holder of the corresponding private key could have produced. The network checks the math. If it passes, the transaction executes. If it doesn't, nothing happens. There is no password reset, no customer support line, no bank to call. The signature is the authorization — full stop.

Bitcoin and Ethereum use ECDSA. Solana and Cardano use EdDSA. Bitcoin Taproot uses Schnorr. These are not arbitrary choices. Each scheme has different mathematical properties that determine whether you can build certain things on top of them — multi-sig aggregation, threshold custody, MPC wallets — without extraordinary complexity.

### The Custody Problem

The largest unsolved operational problem in institutional crypto is key custody. You have a private key worth \$500M. Where do you put it?

A hardware wallet is fine for personal use. For an exchange or a fund, it is a single point of catastrophic failure — one supply chain attack, one rogue engineer, one physical theft. The cold storage arrangements at Mt. Gox, QuadrigaCX, and a dozen other exchanges demonstrate exactly how this ends.

**MPC threshold signing** reframes the problem. Instead of protecting one key, you split the key into $n$ shards distributed across separate machines and parties. Any $t$ of $n$ shards can cooperate to produce a valid signature. No single shard is a complete key. An attacker has to compromise $t$ separate, air-gapped systems simultaneously — which changes the economics of the attack entirely.

| What you're protecting against | How threshold signing helps |
|--------------------------------|-----------------------------|
| Single machine compromised | Attacker gets one shard, which is worthless alone |
| Insider going rogue | Signing requires quorum; no individual can act alone |
| Hardware loss or destruction | Tolerate up to $n - t$ lost shards without losing funds |
| Regulatory audit trail | Every signing event requires documented multi-party cooperation |

Fireblocks, Coinbase Prime, BitGo, and ZenGo all run MPC threshold ECDSA or EdDSA in production today. This is not experimental technology — it is how serious money is moved on blockchains.

> The rest of this document explains the cryptographic machinery underneath: how ECDSA and EdDSA work, why their mathematical properties make MPC harder or easier, and what the leading protocol implementations actually do.



---

## Part 1: Signature Scheme Foundations

> This section exists for one reason: Part 2 is impossible to understand without it. Read it to understand the math, but more importantly, read it to understand *why* ECDSA and Schnorr/EdDSA feel so different to implement in MPC.

---


### 1.1 ECDSA (Elliptic Curve Digital Signature Algorithm)

ECDSA is the default. Bitcoin used it. Ethereum uses it. Most blockchains deployed before 2020 use it. Understanding its signing equation is not optional — and one specific property in that equation, the $k^{-1}$ inversion, is the root cause of everything hard about ECDSA MPC. We'll come back to that in Part 2.

**Interactive Visualization:** https://www.desmos.com/calculator/pxrkkhxcez

#### Variables

| Symbol | Meaning |
|--------|---------|
| $z$ | $\text{hash}(msg)$ — message digest |
| $d$ | private key (scalar) |
| $Q$ | public key — $Q = d \cdot G$ |
| $k$ | random nonce, chosen from $[1, n-1]$ |
| $R$ | random point — $R = k \cdot G$ |
| $r$ | x-coordinate of $R$ |
| $s$ | signature scalar |

#### Signing

$$s = k^{-1} \cdot (z + r \cdot d) \pmod{n}$$

Output signature: $(r,\ s)$

#### Derivation (Why verification works)

It's worth following the derivation carefully — it shows exactly what the verifier can reconstruct without learning the private key. Starting from the signing equation and solving for $k \cdot G$:

$$\begin{aligned}
k \cdot s &= z + r \cdot d \\
k &= \frac{z + r \cdot d}{s} \\
k \cdot G &= \frac{z}{s} \cdot G + \frac{r}{s} \cdot (d \cdot G) \\
&= \frac{z}{s} \cdot G + \frac{r}{s} \cdot Q
\end{aligned}$$

#### Verification

Given $(r, s, z, Q)$, the verifier reconstructs the random point:

$$R' = \frac{z}{s} \cdot G + \frac{r}{s} \cdot Q$$

Valid if $R'.x = r$.

The verifier never sees $k$ or $d$. They only see $r$, $s$, $z$, and $Q$. The math checks out anyway — that's the point.


---

### 1.2 Schnorr Signature

Schnorr is simpler than ECDSA. Not because it's less secure — it's provably at least as secure under the same hardness assumptions — but because the math is additive throughout. There's no modular inversion anywhere in the signing equation. That single difference is why EdDSA MPC is tractable and ECDSA MPC requires Paillier homomorphic encryption.

#### Variables

| Symbol | Meaning |
|--------|---------|
| $a$ | private key (scalar) |
| $A$ | public key — $A = a \cdot G$ |
| $r$ | random nonce scalar |
| $R$ | random point — $R = r \cdot G$ |
| $k$ | challenge — $k = \text{hash}(R,\ A,\ msg)$ |
| $S$ | signature scalar |

#### Signing

$$S = r + k \cdot a$$

Output signature: $(R,\ S)$

#### Derivation (Why verification works)

Multiply both sides by the base point $G$:

$$\begin{aligned}
S \cdot G &= r \cdot G + k \cdot (a \cdot G) \\
&= R + k \cdot A
\end{aligned}$$

#### Verification

Given $(R, S, A)$, the verifier computes $k = \text{hash}(R, A, msg)$ and checks:

$$S \cdot G = R + k \cdot A$$

No division. No inversion. Everything adds. The verifier recomputes $k$ from public information, multiplies, and checks equality. Keep this in mind when we get to MPC.

---

### 1.3 EdDSA (Edwards-curve Digital Signature Algorithm)

EdDSA is Schnorr made concrete, standardized, and deployed at scale. Schnorr is an abstract framework — it specifies *how* to sign but leaves the curve, the hash, and the nonce derivation up to you. EdDSA answers all three questions and closes the door on implementation variation.

**Interactive Visualization:** https://www.desmos.com/calculator/mwnb018xh4

EdDSA (designed by Bernstein et al.) is a signature scheme **mathematically equivalent to Schnorr** but defined independently. It makes three concrete choices that Schnorr leaves open:

| Aspect | Schnorr (abstract) | EdDSA (concrete) |
|--------|--------------------|-----------------|
| Curve | Unspecified | Twisted Edwards curve (e.g., Curve25519) |
| Hash | Unspecified | SHA-512 |
| Nonce | Unspecified | Deterministic: $r = \text{hash}(\text{seed}_{R} \| msg)$ |

The most widely used instantiation is **Ed25519** (EdDSA on Curve25519 with SHA-512).

The deterministic nonce is the most consequential design decision here. In ECDSA, if you reuse $k$ for two messages, an attacker can recover your private key with trivial arithmetic. EdDSA eliminates this failure mode entirely by deriving $r$ from the private seed and the message — the same inputs always produce the same nonce, and you never have to rely on your random number generator being trustworthy.

This is a feature. It also becomes a specific problem in MPC, which we'll address in Section 2.2.

---

### 1.4 Comparison: ECDSA vs. Schnorr vs. EdDSA

The table below is the whole point of Part 1. Everything else in Part 2 follows from the "Linearity" row.

| Property | ECDSA | Schnorr | EdDSA |
|----------|-------|---------|-------|
| Signature equation | $s = k^{-1}(z + rd) \pmod{n}$ | $S = r + k \cdot a$ | $S = r + k \cdot a$ (same as Schnorr) |
| Linearity | Non-linear ($k^{-1}$ inversion) | Linear (additive) | Linear (additive) |
| Nonce type | Random | Random | Deterministic |
| Signature aggregation | Not natively supported | Natively supported | Natively supported |
| MPC complexity | High (requires Paillier HE) | Low (native additive sharing) | Low (native additive sharing) |
| Standardization | Bitcoin (legacy), Ethereum | Bitcoin (Taproot, BIP-340) | TLS, SSH, blockchain wallets |

If you're building MPC on ECDSA, you're fighting the $k^{-1}$ all the way down. If you're building on EdDSA, the math cooperates. Part 2 shows exactly what both of those look like.

---

## Part 2: MPC Applications

Part 1 established the math. Part 2 answers the engineering question: how do you make multiple parties jointly compute a signature when no single party holds the complete key?

The answer is very different depending on which signature scheme you're working with. ECDSA makes this hard in a specific, painful way. EdDSA makes most of it easy — but introduces a different problem around nonce generation that requires its own solution.

---

### 2.1 ECDSA MPC

The following describes the **Lindell 2017** two-party ECDSA MPC protocol (*Fast Secure Two-Party ECDSA Signing*), which is one of the most widely cited constructions. It combines three phases: distributed key generation, nonce generation with Proof of Knowledge, and a Multiplicative-to-Additive (MtA) conversion step that uses Paillier homomorphic encryption to handle ECDSA's non-linear `k^(-1)` term.

> Note: Other ECDSA MPC protocols (e.g., Gennaro-Goldfeder 2018 for threshold $t$-of-$n$, or OT-based MtA variants) share the same high-level structure but differ in the underlying cryptographic primitives.

---

#### Phase 1: Distributed Key Generation (KeyGen)

This is the easy part — relatively speaking. Each party generates a random private key shard. Nobody sends their shard to anyone else. The joint public key is computed from the shards' public counterparts and is mathematically indistinguishable from a key generated on a single machine.

The shards are split **multiplicatively** ($x = x_1 \cdot x_2$), not additively. This feels counterintuitive — additive splitting would be simpler. But ECDSA's signing formula requires computing $k^{-1}$, and modular inverses don't compose cleanly under additive splitting. Multiplicative splitting is what makes the math work out downstream.

$x_1$ and $x_2$ are generated by the respective parties $Q$, without any party learning the full private key $x = x_1 \cdot x_2$:

1. **Alice** generates random $x_1$, computes $Q_1 = x_1 \cdot G$.
2. **Bob** generates random $x_2$, computes $Q_2 = x_2 \cdot G$.
3. Joint public key: $Q = x_1 \cdot Q_2 = x_2 \cdot Q_1 = (x_1 \cdot x_2) \cdot G$.

> Note: In 2-party ECDSA, the key is split **multiplicatively** ($x = x_1 \cdot x_2$), not additively, because the ECDSA signing formula requires $k^{-1}$ — additive splitting would not compose correctly through the inversion.

#### Role of Schnorr: Proof of Knowledge (PoK)

Without this step, the protocol is broken in a specific way. If Alice just broadcasts $Q_1$ without proving she knows $x_1$, Bob could set $Q_2 = Q_{\text{target}} - Q_1$ for any address he wants to control. The joint key would be $Q_{\text{target}}$ — an address only Bob can sign for, because only Bob knows the corresponding scalar.

The fix is elegant: Alice must attach a Schnorr self-signature when she broadcasts $Q_1$. This proves she knows the discrete log of $Q_1$ without revealing $x_1$ itself. Bob verifies:

$$S \cdot G = R + k \cdot Q_1$$

This is a zero-knowledge proof that Alice knows the discrete log of $Q_1$.

---

#### Phase 2: Pre-Signing (Nonce Generation)

Same structure as KeyGen, applied to the signing nonce. Each party picks a random nonce shard, proves they know it (Schnorr PoK again), and the parties compute the aggregate point $R$ without reconstructing the full nonce scalar $k$.

The goal is to compute the nonce $k = k_1 \cdot k_2$ and the aggregate random point $R$ without any party learning the other's nonce shard:

1. **Alice** chooses random $k_1$, computes $R_1 = k_1 \cdot G$.
2. **Bob** chooses random $k_2$, computes $R_2 = k_2 \cdot G$.
3. Aggregate point: $R = k_1 \cdot R_2 = k_2 \cdot R_1 = (k_1 \cdot k_2) \cdot G$; extract $r = R.x$.
4. Schnorr PoK is used again to prove knowledge of each $k_i$.

> Here's the wall: the full signing formula is $s = k^{-1}(z + r \cdot d)$. Expand it with $k = k_1 \cdot k_2$ and $d = x_1 \cdot x_2$, and you get cross-terms like $k_1 \cdot x_2$ — a multiplication between secrets held by different parties. Neither party can compute this without revealing their value. That's the blocker.

---

#### Phase 3: MtA Protocol (Multiplicative-to-Additive Conversion)

Here's where the $k^{-1}$ finally bites. Expand the full signing equation with the multiplicative shares from Phases 1 and 2:

$$s = k^{-1}(z + r \cdot d) \pmod{n}$$

With $k = k_1 \cdot k_2$ (multiplicative nonce sharing) and $d = x_1 \cdot x_2$ (multiplicative key sharing), expanding the full expression reveals why this cannot be computed as additive shares:

$$s = (k_1 k_2)^{-1}(z + r \cdot x_1 x_2)$$

If we denote $k_i^{-1}$ as $\gamma_i$ (so $\gamma_1 \gamma_2 = k^{-1}$), the signing equation becomes:

$$s = \gamma_1 \gamma_2 \cdot z + r \cdot \gamma_1 x_2 \cdot \gamma_2 x_1$$

The terms $\gamma_1 x_2$ and $\gamma_2 x_1$ are **cross-products** — each mixes a secret from Alice with a secret from Bob. Neither party can compute these alone without revealing their secret to the other. This is the fundamental roadblock: **ECDSA requires multiplication across secrets held by different parties.**

Additive sharing fails here because multiplying additive shares produces:

$$(a_1 + a_2)(b_1 + b_2) = a_1 b_1 + a_1 b_2 + a_2 b_1 + a_2 b_2$$

The cross-terms $a_1 b_2$ and $a_2 b_1$ cannot be computed without one party revealing their value to the other.

---

**The solution: Paillier Additive Homomorphic Encryption + MtA**

Paillier encryption is an asymmetric encryption scheme with the special property of **additive homomorphism**:

$$\text{Enc}(a) \cdot \text{Enc}(b) = \text{Enc}(a + b) \qquad \text{(ciphertext multiplication = plaintext addition)}$$

$$\text{Enc}(a)^c = \text{Enc}(c \cdot a) \qquad \text{(ciphertext exponentiation = plaintext scaling)}$$

The **MtA (Multiplicative-to-Additive)** protocol exploits this: given Alice holds secret $a$ and Bob holds secret $b$, MtA converts their product into additive shares $(\alpha, \beta)$ satisfying:

$$\alpha + \beta = a \cdot b \pmod{n}$$

The sketch:

| Step | Who | Action |
|------|-----|---------|
| 1 | Alice | Encrypts her secret: sends $\text{Enc}_{pk_A}(a)$ to Bob |
| 2 | Bob | Computes $\text{Enc}(a)^b = \text{Enc}(a \cdot b)$ using Paillier scalar property; adds random mask $\beta$: sends $\text{Enc}(a \cdot b - \beta)$ to Alice |
| 3 | Alice | Decrypts to get $\alpha = a \cdot b - \beta$; now $\alpha + \beta = a \cdot b$ |

Bob learns nothing about $a$ (encrypted); Alice learns nothing about $b$ (Bob never sends $b$ in the clear). Each cross-product term is converted into a pair of additive shares, and the final signature $s$ becomes expressible as:

$$s = \sigma_1 + \sigma_2 \pmod{n}$$

where $\sigma_1$ and $\sigma_2$ are each computed locally without interaction. The coordinator aggregates them to get the final $s$.

This is not free. The practical costs:

Solving the nonlinearity via Paillier + MtA is mathematically elegant but comes with significant practical costs:

| Cost | Description |
|------|-------------|
| **Key size overhead** | Paillier key sizes are typically 2048–4096 bits (vs. 256-bit for ECC), making keys and ciphertexts large |
| **Computational cost** | Paillier encryption/decryption involves modular exponentiation with large-prime moduli — orders of magnitude slower than ECC operations |
| **Communication rounds** | Each MtA invocation requires 2 interactive rounds between the parties; multiple MtA instances are needed per signing |
| **Range proofs required** | Without additional zero-knowledge proofs (range proofs), the protocol is vulnerable to "small subgroup" and bad-randomness attacks; adding these proofs multiplies proof sizes significantly |
| **No concurrent security without care** | Early constructions (GG18, GG19) lacked proofs under concurrent composition, leading to real-world vulnerabilities |

Linell 2017 worked but was expensive. GG19 broke. CGGMP21 fixed it. The lineage:

| Protocol | Year | Key improvement |
|----------|------|-----------------|
| Lindell 2017 | 2017 | First practical 2-party ECDSA MPC; correct but heavyweight range proofs |
| GG18 / GG19 | 2018–19 | Threshold ($t$-of-$n$); GG19 had flawed range proofs |
| GG20 / Gennaro-Goldfeder | 2020 | Fixed proofs; widely deployed (tss-lib, Binance chain) |
| **CGGMP21** | 2021 | UC-secure, identifiable aborts, key refresh, one-round online signing (with presign phase); state of the art |
| **CGGMP24** | 2024 | Minor revision to CGGMP21; used by **synedrion** and **lfdt-cggmp21** |

> **CGGMP24** is not a separate paper — it is the October 2024 revision of the original [CGGMP21 paper (ePrint 2021/060)](https://eprint.iacr.org/2021/060), which added explicit security checks missing from earlier versions. Both **lfdt-cggmp21** and **synedrion** are based on this 2024 revision.

### 2.2 EdDSA MPC

Because EdDSA inherits Schnorr's linearity, the MPC construction is structurally much simpler. However, one non-obvious challenge arises from EdDSA's deterministic nonce design.

---

#### Why EdDSA is Linearity-Friendly

This is the payoff from Section 1.2. Everything is additive. No Paillier. No MtA. No range proofs. The math just works.

Schnorr/EdDSA satisfies additive homomorphism across both keys and nonces:

$$\begin{aligned}
(a_1 + a_2) \cdot G &= a_1 \cdot G + a_2 \cdot G && \text{(key shares compose additively)} \\
(r_1 + r_2) \cdot G &= r_1 \cdot G + r_2 \cdot G && \text{(nonce shares compose additively)} \\
S_1 + S_2 &= S_{\text{total}} && \text{(partial signatures aggregate directly)}
\end{aligned}$$

No homomorphic encryption is needed — all operations are simply additions modulo the group order. Each party computes their partial signature locally. The coordinator sums them. That's it.

---

#### Why Two Sets of Shares Are Required

SHA-512 is not linear. That single fact forces a specific architectural decision in EdDSA MPC.

EdDSA's private key derivation involves a non-linear step:

$$\text{seed bytes} \xrightarrow{\text{SHA-512}} \text{scalar } a \xrightarrow{\text{clamp}} \text{signing key}$$

To enable additive sharing of the private key $a$, we need:

$$a_1 + a_2 = a_{\text{total}}$$

But SHA-512 is **not linear**, so:

$$\text{hash}(\text{seed}_1) + \text{hash}(\text{seed}_2) \neq \text{hash}(\text{seed}_1 + \text{seed}_2)$$

If we split the seed and each party hashes their share independently, the results don't add up to the correct signing key. You get garbage. The signing will fail verification every time.

The solution is to maintain two separate sets of shares — one for the seed (used for backup and key derivation), and one for the post-hash scalar (used for actual signing). They refer to the same key, but they're stored and operated on separately:

| Share Set | What it represents | Why needed |
|-----------|--------------------|------------|
| **Seed Shares** | Shares of the raw 32-byte seed | Required for backup, recovery, and key derivation |
| **Scalar Shares** | Shares of the post-hash scalar `a` | Required for actual signing (additive, compatible with MPC) |

---

#### The Core Challenge: Deterministic Nonce is Impossible in MPC

Standard single-device EdDSA (RFC 8032) derives the signing nonce deterministically:

$$r = \text{hash}(\text{seed}_R \| msg), \qquad R = r \cdot G$$

EdDSA's deterministic nonce is a feature that becomes a bug in MPC.

On a single device, the nonce is safe because it's deterministic: given the same key and message, you always get the same nonce, so there's nothing to bias or reuse maliciously. In MPC, the complete seed never exists anywhere — that's the whole point. So nobody can compute $\text{seed}_R$, and the deterministic nonce formula is unreachable.

So EdDSA MPC falls back to `RNG()` for nonce generation. This reintroduces the risk of **nonce reuse** — the exact failure mode that deterministic nonces were invented to prevent. Modern production protocols address this with explicit nonce coordination.

---

#### Strategy A: Commit-and-Reveal

Prevent the Last Actor Advantage. If you find out everyone else's $R_i$ values before committing your own, you can pick your $R_i$ to steer the aggregate $R$ wherever you want. The commitment step removes this advantage — you're locked in before you see anyone else's reveal.

The protocol:

| Step | Action |
|------|--------|
| 1. Commit | Each party $i$ computes $r_i$, broadcasts $C_i = \text{hash}(r_i \cdot G)$ |
| 2. Wait | Wait until all commitments $\{C_j\}$ are received |
| 3. Reveal | Broadcast actual $R_i = r_i \cdot G$ |
| 4. Verify | Check $C_i = \text{hash}(R_i)$ for all received values |
| 5. Aggregate | $R = \sum R_i$ |

A stronger variant (Lindell 17, Gennaro-Goldfeder TSS) also requires a Schnorr ZK Proof during the Reveal phase, proving knowledge of the discrete log behind each $R_i$.

---

#### Strategy B: FROST Protocol

**FROST = Flexible Round-Optimized Schnorr Threshold Signatures**

Commit-and-Reveal works, but it's slow — two rounds of network communication every time you sign. FROST solves the performance problem by pre-generating nonce material offline, reducing online signing to a single round. The security challenge it has to solve is: how do you use pre-generated randomness without giving an attacker the ability to manipulate which nonces end up combined?

- Nonces are pre-generated in batch offline, so online signing requires only one round of communication.
- Supports $(t, n)$ threshold — any $t$ of $n$ parties can sign, using Shamir Secret Sharing and Lagrange interpolation.
- Introduces Double Nonce and Binding Factor to prevent the ROS / Wagner's Attack.

The binding factor:

Each party holds two nonce scalars $(d_i, e_i)$ with public commitments $(D_i, E_i) = (d_i \cdot G,\ e_i \cdot G)$. The binding factor ties each party's nonce to the full commitment list and the message — making it impossible to manipulate the aggregate $R$ by choosing which pre-committed nonces to use:

$$\rho_i = \text{hash}\bigl(i,\ \{(D_j, E_j)\}_{j \in \text{signers}},\ msg\bigr)$$

$$R_i = D_i + \rho_i \cdot E_i, \qquad R = \sum_i R_i$$

Note: only $E_i$ is scaled by $\rho_i$; $D_i$ is not. Any attempt to manipulate $D_i$ or $E_i$ changes $\rho_i$ (which commits to all pairs), thereby changing $R$, making pre-planned manipulation impossible.

The signing flow:

| Step | Action |
|------|--------|
| Pre-processing (offline) | Each party generates $(d_i, e_i)$ pairs, publishes $(D_i, E_i)$ commitments |
| Signing setup | Coordinator assembles commitment list for this session |
| Binding factor | $\rho_i = \text{hash}(i,\ \text{commitment list},\ msg)$ for each signer |
| Aggregate nonce | $R_i = D_i + \rho_i \cdot E_i$;   $R = \sum_i R_i$ |
| Partial signatures | $S_i = (d_i + e_i \cdot \rho_i) + \lambda_i \cdot a_i \cdot k$,  where $k = \text{hash}(R, A, msg)$ |
| Aggregation | $S = \sum_i S_i$ |
| Verification | $S \cdot G = R + k \cdot A$ |

---

### 2.3 ECDSA MPC vs. EdDSA MPC: Final Comparison

| Aspect | ECDSA MPC | EdDSA MPC |
|--------|-----------|-----------|
| Underlying scheme | ECDSA (non-linear) | EdDSA / Schnorr (linear) |
| Key sharing | Multiplicative ($x = x_1 \cdot x_2$) | Additive ($a = a_1 + a_2$) |
| Nonce sharing | Multiplicative + MtA conversion | Additive (direct) |
| Homomorphic encryption | Required (Paillier) | Not required |
| Online communication rounds | Multiple (KeyGen + MtA + signing) | Fewer (FROST: 1 online round) |
| Nonce determinism | Can be deterministic per party | Must use RNG (determinism impossible) |
| Nonce security protocol | Schnorr PoK | Commit-and-Reveal or FROST binding factor |
| Implementation complexity | High | Moderate |

---

### 2.4 Code Demo: EdDSA MPC (Simplified)

The following TypeScript implementation demonstrates the EdDSA MPC signing flow using Shamir Secret Sharing (DKG) and Lagrange interpolation (threshold signing). All EC operations are mocked as scalar arithmetic to focus on the MPC logic.

```typescript
import { createHash } from 'crypto';

// Simulate Ed25519 group order L
const L = 7237005577332262213973186563042994240857116359379907606001950938285454250989n;

function mod(n: bigint, m: bigint = L): bigint {
    return ((n % m) + m) % m;
}

// Modular inverse via Fermat's Little Theorem: a^(m-2) mod m
function modInverse(a: bigint, m: bigint = L): bigint {
    let base = mod(a, m);
    let exp = m - 2n;
    let res = 1n;
    while (exp > 0n) {
        if (exp % 2n === 1n) res = (res * base) % m;
        base = (base * base) % m;
        exp /= 2n;
    }
    return res;
}

// Mocked EC point: represents P = val * G (demonstrates additive homomorphism)
// In production, use a proper library such as @noble/ed25519
class MockPoint {
    val: bigint;
    constructor(val: bigint) { this.val = mod(val); }
    static Base() { return new MockPoint(1n); }
    add(other: MockPoint): MockPoint { return new MockPoint(this.val + other.val); }
    mul(scalar: bigint): MockPoint { return new MockPoint(this.val * scalar); }
    toBuffer(): Buffer {
        const hex = this.val.toString(16).padStart(64, '0');
        return Buffer.from(hex, 'hex');
    }
}

function hashToScalar(...inputs: (Buffer | bigint)[]): bigint {
    const hash = createHash('sha512');
    for (const input of inputs) {
        if (typeof input === 'bigint') {
            hash.update(Buffer.from(input.toString(16).padStart(64, '0'), 'hex'));
        } else {
            hash.update(input);
        }
    }
    return mod(BigInt('0x' + hash.digest('hex')), L);
}

class MpcNode {
    id: number;
    threshold: number;
    private secretCoefficients: bigint[] = [];
    public myShare: bigint | null = null;

    constructor(id: number, threshold: number) {
        this.id = id;
        this.threshold = threshold;
    }

    // DKG Phase 1: each node generates a random polynomial f_i(x) of degree t-1
    initKeyGen() {
        this.secretCoefficients = [];
        for (let i = 0; i < this.threshold; i++) {
            this.secretCoefficients.push(BigInt(Math.floor(Math.random() * 1000000)));
        }
        // In production: broadcast commitments C_k = coeff_k * G for verifiability
    }

    // Evaluate f_i(peerId) to produce the share sent to that peer
    getShareFor(peerId: number): bigint {
        let x = BigInt(peerId);
        let y = 0n;
        for (let i = 0; i < this.secretCoefficients.length; i++) {
            y = (y + this.secretCoefficients[i] * (x ** BigInt(i))) % L;
        }
        return y;
    }

    // DKG Phase 2: sum all received shares to form this node's private key shard
    receiveShares(shares: bigint[]) {
        this.myShare = shares.reduce((sum, s) => mod(sum + s), 0n);
        console.log(`[Node ${this.id}] DKG complete. Shard x_${this.id} = ${this.myShare}`);
    }

    prepareNonce(): { r_i: bigint; R_i: MockPoint } {
        const r_i = BigInt(Math.floor(Math.random() * 1000000));
        return { r_i, R_i: MockPoint.Base().mul(r_i) };
    }

    // Compute partial signature S_i = (r_i) + k * (lambda_i * x_i)
    signPart(message: Buffer, globalR: MockPoint, globalPubKey: MockPoint, r_i: bigint, allSignerIds: number[]): bigint {
        if (this.myShare === null) throw new Error("Key not generated");

        // Lagrange coefficient: lambda_i = product((0 - x_j) / (x_i - x_j)) for j != i
        let numerator = 1n;
        let denominator = 1n;
        const x_i = BigInt(this.id);
        for (const sid of allSignerIds) {
            if (sid === this.id) continue;
            const x_j = BigInt(sid);
            numerator   = mod(numerator   * (0n - x_j));
            denominator = mod(denominator * (x_i  - x_j));
        }
        const lambda = mod(numerator * modInverse(denominator));

        // Challenge: k = hash(R || A || M)
        const k = hashToScalar(globalR.toBuffer(), globalPubKey.toBuffer(), message);

        // Partial signature
        return mod(r_i + k * mod(this.myShare * lambda));
    }
}

async function main() {
    // Setup: 3 nodes, threshold = 2 (any 2 of 3 can sign)
    const nodes = [new MpcNode(1, 2), new MpcNode(2, 2), new MpcNode(3, 2)];

    // --- DKG ---
    nodes.forEach(n => n.initKeyGen());

    // Debug: reconstruct full private key (never done in production)
    let globalPrivKeyDebug = nodes.reduce((sum, n) => mod(sum + n['secretCoefficients'][0]), 0n);

    // Distribute shares
    nodes.forEach(receiver => {
        receiver.receiveShares(nodes.map(sender => sender.getShareFor(receiver.id)));
    });

    const globalPublicKey = MockPoint.Base().mul(globalPrivKeyDebug);
    console.log(`\n[Global] Public key A: ${globalPublicKey.val}`);

    // --- MPC Signing (Alice = Node 1, Charlie = Node 3) ---
    const message = Buffer.from("Transfer 100 BTC");
    const signers = [nodes[0], nodes[2]];
    const signerIds = signers.map(n => n.id);

    const nonceData = signers.map(n => n.prepareNonce());

    // Aggregate nonce R = R_1 + R_3
    let globalR = nonceData[0].R_i;
    for (let i = 1; i < nonceData.length; i++) globalR = globalR.add(nonceData[i].R_i);

    // Partial signatures
    const partialS = signers.map((node, idx) =>
        node.signPart(message, globalR, globalPublicKey, nonceData[idx].r_i, signerIds)
    );

    // Aggregate S = S_1 + S_3
    const globalS = partialS.reduce((sum, s) => mod(sum + s), 0n);
    console.log(`\n[Result] Signature: (R=${globalR.val}, S=${globalS})`);

    // --- Verification: S*G == R + k*A ---
    const lhs = MockPoint.Base().mul(globalS);
    const k   = hashToScalar(globalR.toBuffer(), globalPublicKey.toBuffer(), message);
    const rhs = globalR.add(globalPublicKey.mul(k));

    console.log(`LHS (S*G):  ${lhs.val}`);
    console.log(`RHS (R+kA): ${rhs.val}`);
    console.log(lhs.val === rhs.val ? "Verification PASSED." : "Verification FAILED.");
}

main();
```

---

## References

### Papers

[Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
The original practical 2-party ECDSA MPC construction. Introduced MtA with Paillier to handle cross-product terms. Still the clearest exposition of the core problem.

[CGGMP21 — UC Non-Interactive, Proactive, Threshold ECDSA with Identifiable Aborts](https://eprint.iacr.org/2021/060)
Canetti, Gennaro, Goldfeder, Makriyannis, Peled (2021). The current state of the art for threshold ECDSA. UC-secure, supports identifiable aborts, key refresh, and one-round online signing via presigning. The October 2024 revision (CGGMP24) adds explicit security checks that were missing from earlier versions — both `synedrion` and `lfdt-cggmp21` implement this revision.

[FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)
Komlo & Goldberg (2020). The standard threshold Schnorr construction. Early versions were vulnerable to Wagner's Attack / ROS; IETF required a mandatory two-round flow before ratifying the standard.

[IETF Draft — Two-Round Threshold Schnorr Signatures with FROST](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)
The ratified standard based on FROST.

---

### Libraries

[**Taurus multi-party-sig**](https://github.com/taurusgroup/multi-party-sig) — Go — CGGMP21, GG18, FROST
Audited by Kudelski Security. Coinbase identified a vulnerability in 2024; patched. [Advisory](https://github.com/advisories/GHSA-7f6p-phw2-8253).

[**lfdt-cggmp21**](https://github.com/LFDLFoundation/cggmp21) — Rust — CGGMP24
Linux Foundation Decentralized Trust (LFDT) / Lockness project. Production-ready, audited, with CRT and precomputed multi-exponentiation optimizations.

[**synedrion**](https://github.com/nucypher/synedrion) — Rust — CGGMP24
What this repo uses. Built on the `manul` protocol framework. AGPL-3.0. Implements KeyGen, Key Refresh, Presigning, Signing, Interactive Signing, and Key Resharing. Not yet audited — use in production at your own risk.
