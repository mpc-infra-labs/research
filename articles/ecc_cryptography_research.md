# ECC Cryptography & MPC Research Notes


## Part 0: Motivation, Scope, and Roadmap

Every blockchain transaction is, at its core, a math problem. You prove you own an address by producing a signature that only the holder of the corresponding private key could have produced. The network checks the math. If it passes, the transaction executes. If it doesn't, nothing happens. There is no password reset, no customer support line, no bank to call. The signature is the authorization, full stop.

This note is not a general tutorial on elliptic curves, but it also does not assume you're already fluent in ECC notation. The minimum background is small: private keys are scalars, public keys are curve points derived from those scalars, and signature schemes differ in how they combine secrets, nonces, and hashes. Part 1 introduces exactly that much machinery and no more.

The taxonomy also matters. There are three signature schemes in this story, not two. **ECDSA** and **Schnorr** are distinct constructions. **EdDSA** is best understood as a concrete Schnorr-family instantiation with specific choices for curve, hash, and nonce derivation. That distinction is what makes the rest of the note coherent: the MPC story is really "ECDSA versus Schnorr-style signatures," with EdDSA as the most important deployed Schnorr variant.

Bitcoin and Ethereum use ECDSA. Bitcoin Taproot uses Schnorr. Solana and Cardano use EdDSA. These are not arbitrary choices. Each scheme has different mathematical properties that determine whether you can build certain things on top of them — multi-sig aggregation, threshold custody, MPC wallets — without extraordinary complexity.

That is the roadmap of the note. Part 1 starts with a minimal ECC refresher, then walks through ECDSA, Schnorr, and EdDSA in that order. Part 2 shows why ECDSA MPC is hard, why Schnorr keeps reappearing inside those protocols, and why EdDSA inherits Schnorr's nice linearity while introducing its own nonce-coordination problem.

### The Custody Problem

The core operational problem in institutional crypto is not storage. It is custody. A private key can control \$500M. If one machine, one employee, or one recovery procedure can produce a valid signature alone, then your entire security model collapses to that weakest point.

A hardware wallet is fine for an individual. For an exchange, custodian, or treasury operation, it is just a concentrated failure domain. One compromised device, one insider with enough access, one backup flow that looked reasonable on paper, and the funds are gone. Crypto has been relearning the same lesson for a decade: when control reduces to one key, operational mistakes become catastrophic.

**MPC threshold signing** changes the shape of the problem. There is still one logical signing key from the chain's point of view, but no single system ever possesses it outright. Instead, signing power is distributed across $n$ parties, and any $t$ of them can jointly authorize a transaction. A compromised server yields one share, not the key. A rogue operator cannot move funds alone. A lost device is survivable as long as the threshold is still met.

That is the real appeal of threshold cryptography. It does not make key management disappear. It turns a brittle single-point-of-failure problem into a quorum problem. That is a much better trade.

This is why MPC matters in practice. It is not academic ornamentation around signatures. It is the cryptographic machinery that lets institutions keep large balances online without trusting any one machine or any one human being too much.

> The rest of this document explains the cryptographic machinery underneath: the minimal ECC background you need, how ECDSA, Schnorr, and EdDSA relate to each other, why their algebra makes MPC harder or easier, and what the leading protocol implementations actually do.




## Part 1: Signature Scheme Foundations

> This section exists for one reason: Part 2 is impossible to understand without it. It is not a full ECC course. It gives you just enough notation to read the equations, then compares ECDSA, Schnorr, and EdDSA in the order the MPC discussion depends on.

### 1.0 Minimal ECC Refresher

This note only needs four facts about elliptic-curve cryptography:

1. A **scalar** is a number modulo a large group order $n$.
2. A **point** is an element of the elliptic-curve group, and points can be added.
3. **Scalar multiplication** means repeated point addition: $a \cdot G$.
4. A public key is a point $Q = d \cdot G$ derived from a secret scalar $d$; going from $d$ to $Q$ is easy, recovering $d$ from $(G, Q)$ is assumed hard.

That is enough for this note. You do not need the full curve equation, coordinate systems, or point-addition formulas unless you are implementing the curve arithmetic itself.

### 1.1 ECDSA (Elliptic Curve Digital Signature Algorithm)

ECDSA is the default. Bitcoin used it. Ethereum uses it. Most blockchains deployed before 2020 use it. Understanding its signing equation is not optional — and one specific property in that equation, the $k^{-1}$ inversion, is the root cause of everything hard about ECDSA MPC. We'll come back to that in Part 2.

If this is your first time seeing ECDSA notation, read symbols as follows: lowercase letters ($d, k, z, r, s$) are scalars (numbers mod $n$), uppercase letters ($G, Q, R$) are curve points, and $a \cdot G$ means scalar multiplication on the curve (group addition applied $a$ times; geometrically, the point reached by adding $G$ to itself $a$ times on the curve).

#### Notation (Quick Scan)

Curve constants:
- $G$: generator/base point (public constant)
- $n$: order of $G$ (scalar arithmetic is modulo $n$)

Keypair:
- $d$: private key scalar
- $Q = d \cdot G$: public key point

Per-signature quantities:
- $z = \text{hash}(msg)$: message digest scalar
- $k \in [1, n-1]$: fresh nonce scalar
- $R = k \cdot G$: nonce point
- $r = x(R)$: x-coordinate of the nonce point

Signature output:
- $(r, s)$: signature pair

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

#### Interactive Visualization

[click here to try it yourself](https://www.desmos.com/calculator/pxrkkhxcez)

<img src="./assets/ecdsa_demo.gif" alt="ECDSA interactive demo" width="640" />

This is not a literal secp256k1 plot. It is a visual proxy for the algebra.

The demo maps scalar relationships onto the unit circle via $T(t) = (\sin t, \cos t)$ so you can watch signing and verification move together in one frame. The only thing to notice is this: for a valid signature, the verifier reconstructs the same target implied by the signer's nonce. Change $k$ or $z$, and the downstream terms move with it.



### 1.2 Schnorr Signature

Schnorr is simpler than ECDSA. Not because it's less secure — it's provably at least as secure under the same hardness assumptions — but because the math is additive throughout. There's no modular inversion anywhere in the signing equation. That single difference is why EdDSA MPC is tractable and ECDSA MPC requires Paillier homomorphic encryption.

#### Notation (Quick Scan)

Curve constants:
- $G$: generator/base point (public constant)

Keypair:
- $a$: private key scalar
- $A = a \cdot G$: public key point

Per-signature quantities:
- $r$: random nonce scalar
- $R = r \cdot G$: nonce point
- $k = \text{hash}(R, A, msg)$: challenge scalar

Signature output:
- $(R, S)$ where $S = r + k \cdot a$

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


### 1.3 EdDSA (Edwards-curve Digital Signature Algorithm)

EdDSA is Schnorr made concrete, standardized, and deployed at scale. Schnorr is an abstract framework — it specifies *how* to sign but leaves the curve, the hash, and the nonce derivation up to you. EdDSA answers all three questions and closes the door on implementation variation.

#### Notation (Quick Scan)

Curve constants:
- $G$: generator/base point (public constant)

Keypair:
- $a$: private key scalar
- $A = a \cdot G$: public key point

Per-signature quantities:
- $r = \text{hash}(\text{seed}_{R} \| msg)$: deterministic nonce scalar
- $R = r \cdot G$: nonce point
- $k = \text{hash}(R, A, msg)$: challenge scalar

Signature output:
- $(R, S)$ where $S = r + k \cdot a$

#### Interactive Visualization

[click here to try it yourself](https://www.desmos.com/calculator/mwnb018xh4)

<img src="./assets/eddsa_demo.gif" alt="EdDSA interactive demo" width="640" />

This demo visualizes the Ed25519 verification equation $S \cdot G = R + k \cdot A$.

The purple curve is a Twisted Edwards curve. The green point is the public key term $A$, the blue point is the nonce point $R$, the red point is $R + k \cdot A$, and the black point is the verification-side check $S \cdot G$.

In the demo, $k$ is fixed to $1$ for clarity. Drag the private scalar and nonce sliders to watch both sides move together. This is a geometric proxy, not literal Ed25519 field arithmetic.

EdDSA (designed by Bernstein et al.) is a signature scheme **mathematically equivalent to Schnorr** but defined independently. It makes three concrete choices that Schnorr leaves open:

| Aspect | Schnorr (abstract) | EdDSA (concrete) |
|--------|--------------------|-----------------|
| Curve | Unspecified | Twisted Edwards curve (e.g., Curve25519) |
| Hash | Unspecified | SHA-512 |
| Nonce | Unspecified | Deterministic: $r = \text{hash}(\text{seed}_{R} \| msg)$ |

The most widely used instantiation is **Ed25519** (EdDSA on Curve25519 with SHA-512).

The deterministic nonce is the most consequential design decision here. In ECDSA, if you reuse $k$ for two messages, an attacker can recover your private key with trivial arithmetic. EdDSA eliminates this failure mode entirely by deriving $r$ from the private seed and the message — the same inputs always produce the same nonce, and you never have to rely on your random number generator being trustworthy.

This is a feature. It also becomes a specific problem in MPC, which we'll address in Section 2.2.


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


## Part 2: MPC Applications

Part 1 established the math. Part 2 answers the engineering question: how do you make multiple parties jointly compute a signature when no single party holds the complete key?

The answer is very different depending on which signature scheme you're working with. ECDSA makes this hard in a specific, painful way. Schnorr makes the algebra clean. EdDSA inherits that clean structure, but reintroduces complexity through deterministic nonce handling.


### 2.1 ECDSA MPC

The details vary across Lindell 2017, GG20, and CGGMP21. The core problem does not. ECDSA introduces a multiplicative dependency that parties cannot evaluate locally, so every practical protocol has to work around that fact.


#### Phase 1: Distributed Key Generation (KeyGen)

This is the easy part — relatively speaking. Each party generates a random private key shard. Nobody sends their shard to anyone else. The joint public key is computed from the shards' public counterparts and is mathematically indistinguishable from a key generated on a single machine.

The shards are split **multiplicatively** ($x = x_1 \cdot x_2$), not additively. This feels counterintuitive — additive splitting would be simpler. But ECDSA's signing formula requires computing $k^{-1}$, and modular inverses don't compose cleanly under additive splitting. Multiplicative splitting is what makes the math work out downstream.

$x_1$ and $x_2$ are generated by the respective parties $Q$, without any party learning the full private key $x = x_1 \cdot x_2$:

1. **Alice** generates random $x_1$, computes $Q_1 = x_1 \cdot G$.
2. **Bob** generates random $x_2$, computes $Q_2 = x_2 \cdot G$.
3. Joint public key: $Q = x_1 \cdot Q_2 = x_2 \cdot Q_1 = (x_1 \cdot x_2) \cdot G$.

#### Role of Schnorr: Proof of Knowledge (PoK)

Without this step, the protocol is broken in a specific way. If Alice just broadcasts $Q_1$ without proving she knows $x_1$, Bob could set $Q_2 = Q_{\text{target}} - Q_1$ for any address he wants to control. The joint key would be $Q_{\text{target}}$ — an address only Bob can sign for, because only Bob knows the corresponding scalar.

The fix is elegant: Alice must attach a Schnorr self-signature when she broadcasts $Q_1$. This proves she knows the discrete log of $Q_1$ without revealing $x_1$ itself. Bob verifies:

$$S \cdot G = R + k \cdot Q_1$$

This is a zero-knowledge proof that Alice knows the discrete log of $Q_1$.


#### Phase 2: Pre-Signing (Nonce Generation)

Same structure as KeyGen, applied to the signing nonce. Each party picks a random nonce shard, proves they know it (Schnorr PoK again), and the parties compute the aggregate point $R$ without reconstructing the full nonce scalar $k$.

The goal is to compute the nonce $k = k_1 \cdot k_2$ and the aggregate random point $R$ without any party learning the other's nonce shard:

1. **Alice** chooses random $k_1$, computes $R_1 = k_1 \cdot G$.
2. **Bob** chooses random $k_2$, computes $R_2 = k_2 \cdot G$.
3. Aggregate point: $R = k_1 \cdot R_2 = k_2 \cdot R_1 = (k_1 \cdot k_2) \cdot G$; extract $r = R.x$.
4. Schnorr PoK is used again to prove knowledge of each $k_i$.

> Here's the wall: the full signing formula is $s = k^{-1}(z + r \cdot d)$. Expand it with $k = k_1 \cdot k_2$ and $d = x_1 \cdot x_2$, and you get cross-terms like $k_1 \cdot x_2$ — a multiplication between secrets held by different parties. Neither party can compute this without revealing their value. That's the blocker.


#### Phase 3: MtA Protocol (Multiplicative-to-Additive Conversion)

Here's where the $k^{-1}$ finally bites. Expand the signing equation with the multiplicative shares from Phases 1 and 2:

$$s = k^{-1}(z + r \cdot d) \pmod{n}$$

With $k = k_1 \cdot k_2$ and $d = x_1 \cdot x_2$:

$$s = (k_1 k_2)^{-1}(z + r \cdot x_1 x_2)$$

If we denote $k_i^{-1}$ as $\gamma_i$:

$$s = \gamma_1 \gamma_2 \cdot z + r \cdot \gamma_1 x_2 \cdot \gamma_2 x_1$$

The terms $\gamma_1 x_2$ and $\gamma_2 x_1$ are the problem. They are cross-products: each mixes Alice's secret with Bob's secret. Neither side can compute them alone without revealing what it holds. That is the whole story. **ECDSA MPC is hard because ECDSA forces multiplication across secrets owned by different parties.**

Additive sharing fails here because multiplying additive shares produces:

$$(a_1 + a_2)(b_1 + b_2) = a_1 b_1 + a_1 b_2 + a_2 b_1 + a_2 b_2$$

The cross-terms $a_1 b_2$ and $a_2 b_1$ cannot be computed without one party revealing their value to the other.


**The solution: Paillier Additive Homomorphic Encryption + MtA**

This is why practical ECDSA MPC protocols introduce MtA. The goal is not to do the whole signature under homomorphic encryption. It is just to convert a secret product into additive shares without revealing the multiplicands.

Paillier is useful here because it is additively homomorphic:

$$\text{Enc}(a) \cdot \text{Enc}(b) = \text{Enc}(a + b) \qquad \text{(ciphertext multiplication = plaintext addition)}$$

$$\text{Enc}(a)^c = \text{Enc}(c \cdot a) \qquad \text{(ciphertext exponentiation = plaintext scaling)}$$

MtA uses that to turn a product $a \cdot b$ into additive shares $(\alpha, \beta)$ such that:

$$\alpha + \beta = a \cdot b \pmod{n}$$

In sketch form: Alice encrypts $a$, Bob homomorphically mixes in his secret $b$ and a random mask, and Alice decrypts a masked value that becomes her additive share. No one learns the other side's secret, but together they now hold shares of the product.

Once each cross-term is converted this way, the final signature becomes additive again:

$$s = \sigma_1 + \sigma_2 \pmod{n}$$

That works. It is also expensive: large Paillier keys, more communication, more proofs, more edge cases, more ways to get the protocol wrong. That is why the ECDSA line of work evolves from Lindell 2017 through GG20 to CGGMP21/24: everyone is fighting the same non-linearity, then hardening the workaround.

### 2.2 EdDSA MPC

Because EdDSA inherits Schnorr's linearity, the MPC construction is structurally much simpler. However, one non-obvious challenge arises from EdDSA's deterministic nonce design.


#### Why EdDSA is Linearity-Friendly

This is the payoff from Section 1.2. Everything is additive. No Paillier. No MtA. No range proofs. The math just works.

Schnorr/EdDSA satisfies additive homomorphism across both keys and nonces:

$$\begin{aligned}
(a_1 + a_2) \cdot G &= a_1 \cdot G + a_2 \cdot G && \text{(key shares compose additively)} \\
(r_1 + r_2) \cdot G &= r_1 \cdot G + r_2 \cdot G && \text{(nonce shares compose additively)} \\
S_1 + S_2 &= S_{\text{total}} && \text{(partial signatures aggregate directly)}
\end{aligned}$$

No homomorphic encryption is needed — all operations are simply additions modulo the group order. Each party computes their partial signature locally. The coordinator sums them. That's it.


#### The Core Challenge: Deterministic Nonce Does Not Survive MPC

Standard single-device EdDSA (RFC 8032) derives the signing nonce deterministically:

$$r = \text{hash}(\text{seed}_R \| msg), \qquad R = r \cdot G$$

On one machine, this is a feature. It removes RNG fragility and prevents catastrophic nonce reuse. In MPC, the complete seed is never reconstructed anywhere. That is the whole point of threshold custody. So nobody can evaluate the standard deterministic nonce function directly.

This creates the main practical wrinkle in EdDSA MPC. The signature equation is linear, but the standard nonce derivation is not threshold-friendly. Real protocols therefore replace single-device determinism with coordinated randomness: commit-and-reveal in simpler constructions, or FROST-style preprocessed nonces in faster ones.


#### Strategy A: Commit-and-Reveal

The first fix is simple: if parties must use fresh randomness, make sure nobody can wait, see everyone else's nonce point, and then choose theirs last.

| Step | Action |
|------|--------|
| 1. Commit | Each party samples $r_i$ and broadcasts a commitment to $R_i = r_i \cdot G$ |
| 2. Wait | Everyone collects all commitments |
| 3. Reveal | Each party reveals $R_i$ |
| 4. Verify | Check each reveal matches its earlier commitment |
| 5. Aggregate | Set $R = \sum_i R_i$ |

That removes the last-actor advantage. A stronger variant also attaches a Schnorr proof showing each party actually knows the discrete log behind the revealed $R_i$.


#### Strategy B: FROST

Commit-and-reveal works, but it costs an extra online round every time you sign. FROST moves most of that work offline. Parties pre-generate nonce material, publish commitments in advance, and then use a binding factor to tie each signer's nonce contribution to the full commitment list and the message.

Each signer has nonce commitments $(D_i, E_i)$ and computes:

$$\rho_i = \text{hash}\bigl(i,\ \{(D_j, E_j)\}_{j \in \text{signers}},\ msg\bigr)$$

$$R_i = D_i + \rho_i \cdot E_i, \qquad R = \sum_i R_i$$

The point is not the notation. The point is that nonce selection is now bound to the entire signing session, so an attacker cannot game the aggregate $R$ by choosing which preprocessed nonces to combine after seeing partial information.


#### Implementation Note: Two Sets of Shares

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
| **Scalar Shares** | Shares of the post-hash scalar $a$ | Required for actual signing (additive, compatible with MPC) |


### 2.3 Bottom Line

The contrast is now simple. ECDSA fights you at the algebra level, so threshold ECDSA needs heavy machinery just to evaluate the signing equation safely. Schnorr-style signatures cooperate algebraically, so threshold signing is much cleaner. EdDSA inherits that benefit, but gives back some complexity at the nonce layer because the standard deterministic construction does not survive distribution across parties.

That is why the modern protocol landscape looks the way it does. Threshold ECDSA is dominated by MtA-, Paillier-, and proof-heavy constructions such as CGGMP. Threshold Schnorr is dominated by simpler designs such as FROST. Same custody problem. Very different cryptographic ergonomics.


## Appendix: Toy Threshold Schnorr Demo (Scalar-Only)

The following TypeScript is not a production EdDSA MPC implementation. It is a toy threshold-Schnorr sketch over scalar arithmetic. That is exactly why it is useful: it makes the additive structure visible without dragging in real curve arithmetic, deterministic nonce replacement, FROST binding factors, or malicious-party protections.

If you read only one thing from this appendix, it should be this: once the signature equation is additive, threshold signing starts to look mechanically straightforward. The rest of the engineering difficulty comes from making that simple algebra safe in the real world.

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
Canetti, Gennaro, Goldfeder, Makriyannis, Peled (2021). The current state of the art for threshold ECDSA. UC-secure, supports identifiable aborts, key refresh, and one-round online signing via presigning. The October 2024 revision (CGGMP24) adds explicit security checks that were missing from earlier versions — both synedrion and lfdt-cggmp21 implement this revision.

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
What this repo uses. Built on the manul protocol framework. AGPL-3.0. Implements KeyGen, Key Refresh, Presigning, Signing, Interactive Signing, and Key Resharing. Not yet audited — use in production at your own risk.
