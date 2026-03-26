# ECC Cryptography Research Notes

## 1. ECDSA (Elliptic Curve Digital Signature Algorithm)

**Interactive Visualization:** https://www.desmos.com/calculator/pxrkkhxcez

### Core Formula

| Variable | Meaning |
|----------|---------|
| `z` | hash(msg) — message hash |
| `d` | private key |
| `k` | a random integer chosen from [1, n-1] |
| `r` | x-coordinate of the random point R |

**Signing equation:**

```
s = k^(-1) * (z + r*d)  mod n
```

**Derivation:**

```
k * s = z + r*d
=> k = (z + r*d) / s
=> k*G = (z/s)*G + (r/s)*(d*G)
       = (z/s)*G + (r/s)*Q
```

### Verification (Validator's Perspective)

Known information:
- `r` — x-coordinate of the random point
- `s` — signature scalar
- `z` — hash(msg)
- `Q` — public key

The validator can recover the random point via:

```
R' = (z/s)*G + (r/s)*Q
```

Verify that `R'.x == r`.

---

## 2. Schnorr Signature

### Core Formula

| Variable | Meaning |
|----------|---------|
| `a` | private key |
| `R` | random point (`r*G`) |
| `k` | hash(R, A, msg) — the challenge |

**Signing equation:**

```
S = r + k*a
```

**Derivation:**

```
S*G = r*G + k*(a*G)
    = R + k*A
```

### Verification (Validator's Perspective)

Known information:
- `R` — random point
- `S` — signature scalar
- `k` — challenge hash
- `A` — public key

Verify:

```
S*G == R + k*A
```

---

## 3. ECDSA vs. Schnorr Comparison

| Property | ECDSA | Schnorr |
|----------|-------|---------|
| Linearity | Non-linear (uses division/inverse) | Linear (purely additive) |
| MPC Compatibility | Requires Paillier HE for MPC | Natively supports MPC |
| Signature Aggregation | Not supported natively | Natively supported |
| Standardization | Widely deployed (Bitcoin legacy, Ethereum) | Newer (Bitcoin Taproot, EdDSA) |

---

## 4. ECDSA MPC — Combining ECDSA, Paillier & Schnorr

### Phase 1: Distributed Key Generation (KeyGen)

**Goal:** Each participant generates their own private key shard. The joint public key `Q` is computed. No single party knows the full private key `x`.

1. **Alice** generates random `x_1`, computes public key shard `Q_1 = x_1 * G`.
2. **Bob** generates random `x_2`, computes public key shard `Q_2 = x_2 * G`.
3. They exchange `Q_1` and `Q_2`.
4. Joint public key: `Q = Q_1 + Q_2`

#### Role of Schnorr: Proof of Knowledge (PoK)

**Protocol requirement:** When Alice broadcasts `Q_1`, she must attach a Schnorr self-signature.

**Proof statement:** "I prove that I know the `x_1` behind `Q_1`."

**Concrete math:**
- Alice constructs a standard Schnorr signature `(R, S)` using `x_1` as the signing key.
- Bob verifies: `S*G == R + k*Q_1`

This prevents any party from setting their public key shard to something malicious (e.g., `Q_2 - Q_1` to zero out another's contribution).

---

### Phase 2: Pre-Signing (Nonce Generation)

**Goal:** Generate the random nonce `k` shards for ECDSA and aggregate the random point `R`.

1. **Alice** chooses random `k_1`, computes `R_1 = k_1 * G`.
2. **Bob** chooses random `k_2`, computes `R_2 = k_2 * G`.
3. Aggregate: `R = R_1 + R_2`; extract `r = R.x`.
4. Schnorr is used again for Proof of Knowledge on each `k_i`.

> **Important:** In ECDSA MPC, `k` is split **multiplicatively** (`k = k_1 * k_2`), not additively. This is because ECDSA requires `k^(-1)`, and multiplicative sharing is what makes the MtA conversion necessary. Each party holds `k_1^(-1)` and `k_2^(-1)` respectively, and the product `k_1^(-1) * k_2^(-1) = k^(-1)` can then be expressed as additive shares via MtA.

---

### Phase 3: MtA Protocol (Multiplicative to Additive)

**Core insight of ECDSA MPC:** Use Homomorphic Encryption (HE) to convert multiplication/division into addition.

**Challenge:**

The ECDSA signing formula involves `k^(-1)`:

```
s = k^(-1) * (z + r*d)
```

The product `k_1 * k_2` is multiplicative, but:

```
(k_1 + k_2)^(-1) != k_1^(-1) + k_2^(-1)
```

So simple additive sharing does not work.

**Solution: Paillier Additive Homomorphic Encryption**

Paillier encryption has the property:
```
Enc(a) * Enc(b) = Enc(a + b)    (additive homomorphism: ciphertext multiplication = plaintext addition)
Enc(a)^c       = Enc(c*a)       (scalar multiplication: ciphertext exponentiation = plaintext scaling)
```

MtA uses this to allow two parties holding multiplicative shares `a` and `b` to arrive at additive shares `alpha` and `beta` where:

```
alpha + beta = a * b  (mod n)
```

This allows converting multiplicative nonce sharing into an additive form compatible with ECDSA's final aggregation step.

---

## 5. EdDSA (Edwards-curve Digital Signature Algorithm)

**Interactive Visualization:** https://www.desmos.com/calculator/mwnb018xh4

EdDSA (designed by Bernstein et al.) is a signature scheme that is **mathematically equivalent to Schnorr** but defined independently. It operates on a **Twisted Edwards Curve** with specific parameter choices.

- **Schnorr** defines *how to compute*: the signing equation `S = r + k*a`
- **EdDSA** specifies *where to compute* (Twisted Edwards curve, e.g., Curve25519) and *what hash to use* (SHA-512), along with a deterministic nonce derivation

The most widely used variant is **Ed25519**, operating on Curve25519.

---

## 6. EdDSA MPC

### Why EdDSA is Naturally MPC-Friendly

Because EdDSA is a member of the Schnorr family, it inherits Schnorr's **linearity** property:

```
(a_1 + a_2) * G = a_1*G + a_2*G      (key linearity)
(r_1 + r_2) * G = r_1*G + r_2*G      (nonce linearity)
(S_1 + S_2)     = S_total             (signature aggregation)
```

This means all arithmetic is additive — no need for complex HE like Paillier.

---

### Why EdDSA MPC Requires Two Sets of Shares

**Share Set 1 — Seed Shares:** Corresponds to the original EdDSA `Seed` (the raw 32-byte random number).

**Share Set 2 — Scalar Shares:** Corresponds to the `Private Scalar a`. This is derived from `Hash(Seed)` followed by the **clamp** operation (bit manipulation).

**The fundamental problem:** To satisfy Schnorr linearity, the private key must satisfy:

```
a_1 + a_2 = a_total
```

But SHA-512 is NOT linear:

```
Hash(seed_1) + Hash(seed_2) != Hash(seed_1 + seed_2)
```

Therefore, we cannot split at the seed level and then hash — the result would be inconsistent. We must maintain a separate set of shares for the post-hash scalar `a` to enable correct MPC signing.

---

### The Core Challenge: Commitment Generation

**Standard single-device EdDSA (RFC 8032) nonce commitment:**

```
r = Hash(right_half_of_seed || message)
R = r * G
```

**The MPC problem:** This `r` depends on `master_seed`. Since the entire premise of MPC is that no single party ever holds or reconstructs the complete seed, nobody can compute `right_half_of_seed`. Therefore, the standard deterministic nonce commitment is **impossible** in MPC.

**Solution:** In MPC, we are forced to abandon the deterministic nonce (Deterministic Nonce) and instead use `RNG()`. However, this introduces a new vulnerability: **nonce reuse attacks**. Modern production-grade MPC protocols (e.g., Fireblocks, ZenGo) mitigate this with additional hardening strategies.

---

### Interaction Strategy A: Commit-and-Reveal

**Purpose:** Prevent the "Last Actor Advantage" — an adversary who sees all other participants' `R_i` values last could craft their own `R_i` to bias the aggregate `R`.

**Protocol:**

1. **Commit:** Each party `i` computes their nonce `r_i`, and broadcasts `C_i = Hash(r_i * G)` to all others.
2. **Wait:** Each participant waits until they have received commitments `C` from all other participants.
3. **Reveal:** Once all commitments are collected, each party broadcasts their actual `R_i = r_i * G`.
4. **Verify:** Each party verifies `C_i == Hash(R_i)` for all received values.
5. **Aggregate:** `R = sum(R_i)`

**Stronger variant:** More robust MPC protocols (e.g., based on Lindell 17 or Gennaro-Goldfeder TSS) require a Schnorr ZK Proof to be attached during the Reveal phase, proving knowledge of the discrete log behind each `R_i`.

---

### Interaction Strategy B: FROST Protocol

**FROST = Flexible Round-Optimized Schnorr Threshold Signatures**

**Performance characteristics:**
- Batch pre-processes and generates many nonces in advance.
- Only requires **one round** of online communication for signing (no Commit-and-Reveal overhead).

**Security design:**
- Introduces **Double Nonce** and a **Binding Factor** to prevent the ROS (Random Oracle Simulation) / Wagner's Attack.

#### How the Binding Factor Works

In FROST, each party holds **two nonce scalars** `(d_i, e_i)` with corresponding public commitments `(D_i, E_i) = (d_i*G, e_i*G)`. The binding factor ties each party's nonce to the full commitment list and the message:

```
rho_i = Hash(i, {(D_j, E_j) for all j in signers}, message)
R_i   = D_i + rho_i * E_i          (each party's combined nonce point)
R     = sum(R_i)                    (aggregate nonce)
```

Note that `D_i` is **not** multiplied by `rho_i` — only `E_i` is scaled. This creates a cryptographic lock: any attempt to manipulate `D_i` or `E_i` automatically changes `rho_i` (since it commits to all `(D_j, E_j)`), which changes `R_i`, thereby nullifying the attack.

**Flow:**

| Step | Action |
|------|--------|
| Pre-processing | Each party generates many `(d_i, e_i)` scalars, publishes `(D_i, E_i) = (d_i*G, e_i*G)` |
| Signing setup | Coordinator collects the commitment list for this signing session |
| Binding factor | `rho_i = Hash(i, commitment_list, msg)` for each signer |
| Aggregate nonce | `R_i = D_i + rho_i * E_i`; `R = sum(R_i)` |
| Partial signatures | Each party computes `S_i = (d_i + e_i * rho_i) + lambda_i * a_i * k` |
| Aggregation | `S = sum(S_i)` |
| Verification | `S * G == R + k * A` |

---

## 7. EdDSA MPC vs. ECDSA MPC Comparison

| Aspect | EdDSA MPC | ECDSA MPC |
|--------|-----------|-----------|
| Underlying signature | Schnorr (linear) | ECDSA (non-linear) |
| Homomorphic encryption needed | No | Yes (Paillier) |
| Communication rounds | Fewer (FROST: 1 online round) | More (MtA requires multiple rounds) |
| Nonce generation | Must use RNG (not deterministic) | Must use MtA for k^(-1) |
| Key linearity | Naturally additive | Requires HE conversion |
| Implementation complexity | Lower | Higher |

---

## 8. EdDSA MPC Code Demo

A simplified TypeScript demonstration of the EdDSA MPC signing flow using Shamir Secret Sharing and Lagrange interpolation:

```typescript
import { createHash } from 'crypto';

// Simulate Ed25519 group order L using a large prime
const L = 7237005577332262213973186563042994240857116359379907606001950938285454250989n;

// Modular arithmetic (handles negatives)
function mod(n: bigint, m: bigint = L): bigint {
    return ((n % m) + m) % m;
}

// Modular inverse via Fermat's Little Theorem: a^(m-2) mod m
// Used to compute Lagrange coefficients (division)
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

// Simulated elliptic curve point operations
// In real code, use a proper EC library (e.g., @noble/ed25519)
class MockPoint {
    val: bigint; // Represents a point as P = val * G (for demonstrating homomorphism)

    constructor(val: bigint) { this.val = mod(val); }

    static Base() { return new MockPoint(1n); } // Base point G

    // Point addition: P1 + P2
    add(other: MockPoint): MockPoint {
        return new MockPoint(this.val + other.val);
    }

    // Scalar multiplication: s * P
    mul(scalar: bigint): MockPoint {
        return new MockPoint(this.val * scalar);
    }

    // Encoding (serialization)
    toBuffer(): Buffer {
        const hex = this.val.toString(16).padStart(64, '0');
        return Buffer.from(hex, 'hex');
    }

    static fromBuffer(buf: Buffer): MockPoint {
        return new MockPoint(BigInt('0x' + buf.toString('hex')));
    }
}

// Hash helper: maps arbitrary inputs to a scalar mod L
function hashToScalar(...inputs: (Buffer | bigint)[]): bigint {
    const hash = createHash('sha512');
    for (const input of inputs) {
        if (typeof input === 'bigint') {
            const hex = input.toString(16).padStart(64, '0');
            hash.update(Buffer.from(hex, 'hex'));
        } else {
            hash.update(input);
        }
    }
    return mod(BigInt('0x' + hash.digest('hex')), L);
}

class MpcNode {
    id: number;
    threshold: number;

    // DKG temporary state
    private secretCoefficients: bigint[] = []; // Coefficients of f_i(x)

    // DKG results
    public myShare: bigint | null = null;       // x_i (private key shard)
    public publicKeyShares: Map<number, MockPoint> = new Map();

    constructor(id: number, threshold: number) {
        this.id = id;
        this.threshold = threshold;
    }

    // --- DKG Step 1: Generate polynomial ---
    initKeyGen() {
        // Generate t random coefficients; the constant term is the party's secret u_i
        this.secretCoefficients = [];
        for (let i = 0; i < this.threshold; i++) {
            this.secretCoefficients.push(BigInt(Math.floor(Math.random() * 1000000)));
        }
        // In a real protocol, broadcast commitments: C_k = coeff_k * G
    }

    // Compute share for peerId: evaluate f_i(peerId)
    getShareFor(peerId: number): bigint {
        let x = BigInt(peerId);
        let y = 0n;
        for (let i = 0; i < this.secretCoefficients.length; i++) {
            let term = (this.secretCoefficients[i] * (x ** BigInt(i))) % L;
            y = (y + term) % L;
        }
        return y;
    }

    // --- DKG Step 2: Receive shares and combine into private key shard ---
    receiveShares(shares: bigint[]) {
        let sum = 0n;
        for (const s of shares) {
            sum = mod(sum + s);
        }
        this.myShare = sum;
        console.log(`[Node ${this.id}] DKG complete. My private key shard x_${this.id} = ${this.myShare}`);
    }

    // --- Signing Step 1: Generate nonce ---
    prepareNonce(): { r_i: bigint, R_i: MockPoint } {
        const r_i = BigInt(Math.floor(Math.random() * 1000000));
        const R_i = MockPoint.Base().mul(r_i);
        return { r_i, R_i };
    }

    // --- Signing Step 2: Compute partial signature S_i ---
    // S_i = r_i + k * (x_i * lambda_i)
    signPart(
        message: Buffer,
        globalR: MockPoint,
        globalPubKey: MockPoint,
        my_r_i: bigint,
        allSignerIds: number[]
    ): bigint {
        if (this.myShare === null) throw new Error("Key not generated");

        // 1. Compute Lagrange coefficient for this signer
        // lambda_i = product( (0 - x_j) / (x_i - x_j) ) for j in signers, j != i
        let numerator = 1n;
        let denominator = 1n;
        const x_i = BigInt(this.id);

        for (const signerId of allSignerIds) {
            if (signerId === this.id) continue;
            const x_j = BigInt(signerId);
            numerator = mod(numerator * (0n - x_j));
            denominator = mod(denominator * (x_i - x_j));
        }

        const lambda = mod(numerator * modInverse(denominator));
        console.log(`[Node ${this.id}] Lagrange weight for signers [${allSignerIds}]: lambda = ${lambda}`);

        // 2. Compute challenge k = Hash(R || A || M)
        const k = hashToScalar(globalR.toBuffer(), globalPubKey.toBuffer(), message);

        // 3. Compute partial signature S_i = r_i + k * (x_i * lambda_i)
        const weightedPrivKey = mod(this.myShare * lambda);
        const s_i = mod(my_r_i + k * weightedPrivKey);

        return s_i;
    }
}

async function main() {
    console.log("=== 1. Initialize 3 Nodes (Threshold = 2) ===");
    const nodes = [
        new MpcNode(1, 2), // Alice
        new MpcNode(2, 2), // Bob
        new MpcNode(3, 2)  // Charlie
    ];

    // --- Phase 1: DKG ---
    console.log("\n=== 2. Execute DKG (Distributed Key Generation) ===");

    // Each node generates their polynomial
    nodes.forEach(n => n.initKeyGen());

    // Distribute shares (simulating network communication)
    let globalPrivKeyDebug = 0n; // Debug only; in real world, nobody knows this
    nodes.forEach(n => globalPrivKeyDebug = mod(globalPrivKeyDebug + n['secretCoefficients'][0]));

    nodes.forEach(receiver => {
        const received = nodes.map(sender => sender.getShareFor(receiver.id));
        receiver.receiveShares(received);
    });

    // Compute joint public key (A = s * G)
    const globalPublicKey = MockPoint.Base().mul(globalPrivKeyDebug);
    console.log(`\n[Global] Joint public key A computed: ${globalPublicKey.val} (simulated value)`);

    // --- Phase 2: MPC Signing ---
    console.log("\n=== 3. Execute MPC Signing (Alice and Charlie) ===");

    const message = Buffer.from("Transfer 100 BTC");
    const signers = [nodes[0], nodes[2]]; // Node 1 (Alice) and Node 3 (Charlie)
    const signerIds = signers.map(n => n.id);

    // Step 1: Each signer generates their nonce r_i
    console.log("-> Step 1: Generate nonces r_i");
    const nonceData = signers.map(n => n.prepareNonce());

    // Step 2: Aggregate nonce points R = R_1 + R_3
    let globalR = nonceData[0].R_i;
    for (let i = 1; i < nonceData.length; i++) {
        globalR = globalR.add(nonceData[i].R_i);
    }
    console.log(`[Coordinator] Aggregated nonce point R: ${globalR.val}`);

    // Step 3: Each signer computes their partial signature S_i
    console.log("-> Step 2: Compute partial signatures S_i (with Lagrange weights)");
    const partialS = signers.map((node, idx) => {
        return node.signPart(
            message,
            globalR,
            globalPublicKey,
            nonceData[idx].r_i,
            signerIds
        );
    });

    // Step 4: Aggregate S = S_1 + S_3
    console.log("-> Step 3: Aggregate S");
    let globalS = 0n;
    for (const s_i of partialS) {
        globalS = mod(globalS + s_i);
    }

    const signature = { R: globalR, S: globalS };
    console.log(`\n[Result] Final signature: (R=${signature.R.val}, S=${signature.S})`);

    // --- Phase 3: Verification ---
    console.log("\n=== 4. Verify Signature (Standard EdDSA) ===");
    // Verification equation: S * G == R + k * A

    // Left side: S * G
    const lhs = MockPoint.Base().mul(signature.S);

    // Right side: R + k * A
    const k = hashToScalar(signature.R.toBuffer(), globalPublicKey.toBuffer(), message);
    const rhs = signature.R.add(globalPublicKey.mul(k));

    console.log(`Verification LHS (S*G): ${lhs.val}`);
    console.log(`Verification RHS (R+kA): ${rhs.val}`);

    if (lhs.val === rhs.val) {
        console.log("Verification PASSED! Signature is valid.");
    } else {
        console.error("Verification FAILED!");
    }
}

main();
```

---

## 9. References & Further Reading

### FROST Protocol

- **Original Paper:** [Gemini conversation on FROST](https://gemini.google.com/app/a124e9123f1513c9)
- **Historical Note:** In FROST's early versions, it was vulnerable to **Wagner's Attack / ROS (Random Oracle Simulation)**. IETF subsequently mandated a two-round protocol (Commit + Reveal) in its standardization.

### IETF Standard

- **Title:** Two-Round Threshold Schnorr Signatures with FROST
- **Link:** https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/

### Production Implementations

- **Taurus Group (multi-party-sig)**
  - Audited by Kudelski Security
  - Reviewed by Coinbase in 2024; issues have been fixed
  - Security advisory: https://github.com/advisories/GHSA-7f6p-phw2-8253
