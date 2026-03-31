# If No One Has the Key, Who Signs?

*MPC wallets promise distributed trust. Blockchains still demand an ordinary signature. The gap between those two facts is why threshold ECDSA feels so much heavier than Schnorr.*

When a wallet vendor says it uses MPC, the usual promise is straightforward enough: no single machine, employee, or compromised server can move funds alone.

On that framing, the job looks easy: split the key into pieces, distribute them across different systems, require a quorum, and call it a day.

In practice, though, "MPC wallet" can describe very different architectures, trust assumptions, and failure modes. The label tells you much less than it seems to.

But a blockchain does not accept "distributed trust" as an output. It accepts a normal signature. If a chain expects an ECDSA signature, your distributed system still has to produce a perfectly ordinary ECDSA signature without ever reconstructing the private key in one place.

That is where MPC wallets stop being a storage story and become a cryptographic engineering problem.

Even if you never plan to build a custody company, this helps explain why wallet infrastructure is expensive, why some security architectures are harder to audit from the outside than they first appear, and why the same phrase, "MPC wallet," can hide radically different amounts of cryptographic machinery underneath.

It also explains why the underlying chain matters. Ethereum still relies on ECDSA. Bitcoin added Schnorr with Taproot, though much of its wallet infrastructure still lives in the older ECDSA world. Solana uses Ed25519, a Schnorr-family design. Those are not cosmetic differences. They change how naturally signing can be distributed across multiple parties.

> Threshold signing is not equally hard across signature schemes. ECDSA is painful because its signing equation is non-linear. Schnorr-style signatures are cleaner because their signing equation stays additive.

That one distinction explains more of the wallet and custody landscape than it first seems like it should.

## MPC Changes the Authorization Boundary

Wallet security is often framed as a storage problem: where is the private key kept, and how do you keep someone from stealing it?

For serious custody systems, the problem is better described as an authorization problem. A private key can control an enormous amount of value. If one device, one admin, or one fallback process can produce a valid signature alone, then the real security boundary is just that weakest point.

Threshold signing changes that boundary.

It changes where trust sits. It does not eliminate governance failures, client compromise, insider abuse, or bad operational policy.

From the chain's point of view, there is still one public key and one valid signature. But behind the scenes, signing power is distributed across multiple parties, and no one party ever holds the complete signing key outright.

This is different from ordinary multisig. Multisig changes what the chain sees. It exposes multiple keys, explicit script structure, or explicit policy logic on-chain. Threshold signing tries to preserve the external appearance of an ordinary wallet while distributing trust off-chain.

That flexibility is an operational advantage. It also makes the real trust boundary harder for outsiders to inspect.

This is the infrastructure underneath exchanges, custodians, treasury systems, and wallet products that want stronger security without turning every transaction into a visibly unusual on-chain construction. It is one reason some wallet systems are more expensive, slower to ship, harder to audit, or more operationally fragile than others.

## The Relevant Technical Fact

Most of elliptic-curve cryptography can stay off the page. Only one part matters here.

- A private key is a secret number.
- A public key is derived from it.
- A signature scheme specifies how a private key, a message, and a nonce combine to produce a valid signature.

For threshold signing, that last point is the whole story.

If the signature equation is additive, parties can often hold additive shares and combine their local work cleanly.

If the signature equation forces multiplication across secrets held by different parties, life gets worse quickly.

That is the difference between "this is an elegant protocol" and "this is a large cryptographic engineering project."

## The Signature Scheme Choice Is Not Cosmetic

For custody infrastructure, three schemes matter most:

- `ECDSA`, because Ethereum and much of today's Bitcoin infrastructure still run on it
- `Schnorr`, because it is the cleaner algebraic alternative, including Bitcoin's Taproot signatures
- `EdDSA`, because it is the most important deployed Schnorr-family construction, especially through Ed25519 on systems such as Solana

For threshold signing, the practical takeaway is simpler:

- `ECDSA` is the awkward one for MPC
- `Schnorr` is the clean one
- `EdDSA` mostly inherits Schnorr's clean structure, but adds a nonce-coordination wrinkle

The equations make the contrast visible immediately.

ECDSA:

$$s = k^{-1}(z + r d) \pmod{n}$$

Schnorr:

$$S = r + c a$$

EdDSA:

$$S = r + c a$$

The key term is the ECDSA inverse:

$$k^{-1}$$

That term is the biggest algebraic reason threshold ECDSA gets unpleasant.

In Schnorr-style signatures, the algebra stays additive from end to end.

That structural difference is the whole story.

## Why ECDSA Makes MPC So Heavy

Threshold signing initially looks like a secret-sharing problem.

Take a private key. Split it across several parties. Have each party do its part. Combine the results. Return one valid signature.

That intuition is close enough to be dangerous.

ECDSA does not only require secret values. It requires a computation over those secret values that does not naturally decompose across parties.

The signing equation is:

$$s = k^{-1}(z + r d) \pmod{n}$$

The private key `d` is secret. The nonce `k` is secret. In a threshold setting, different parties hold different shares of both.

Again, use a toy two-party split just for intuition:

$$d = d_1 + d_2,\qquad k = k_1 + k_2$$

Substitute those shares into the ECDSA equation:

$$s = (k_1 + k_2)^{-1}\bigl(z + r(d_1 + d_2)\bigr)$$

And expand the inner term:

$$s = (k_1 + k_2)^{-1}(z + r d_1 + r d_2)$$

This is exactly where the clean decomposition breaks. In the Schnorr case, the sum of shares stayed a sum all the way through. Here the parties run straight into the inverse of a shared secret nonce:

$$ (k_1 + k_2)^{-1} $$

That object does not split into local pieces in any useful way. In particular:

$$ (k_1 + k_2)^{-1} \neq k_1^{-1} + k_2^{-1} $$

So there is no equally simple rewrite of the form:

$$s = s_1 + s_2$$

with each party computing its own local term from only its own share. The trouble is not merely that `d` and `k` are secret. The trouble is that ECDSA asks the parties to compute across those secrets in a way that does not stay linear.

Now look at the shape of the computation:

- invert a secret nonce
- mix that with the message hash
- multiply the public nonce component `r` by the secret signing key

The problem is not just secrecy. The problem is that the computation is not naturally share-friendly.

In practice, threshold ECDSA turns into a secure multiplication problem. Parties need the effect of multiplying secrets they do not want to reveal to one another.

That is why threshold ECDSA protocols accumulate extra machinery:

- Paillier-based homomorphic encryption
- MtA, or multiplicative-to-additive conversion
- zero-knowledge proofs and other checks to make sure parties followed the protocol correctly

None of that is decorative. It is compensation for an algebraic problem.

And those algebraic costs do not stay in the math layer. They show up as more rounds, more presigning queues, more proof machinery, more implementation surface, and more ways for a production system to become slow, brittle, or hard to audit.

This is why people who work on threshold ECDSA sound slightly haunted. They are not being dramatic. They are describing a protocol family that fights the signature equation the whole way through.

## Why Schnorr Is Cleaner

Schnorr has a different shape:

$$S = r + c a$$

Everything important is additive.

Suppose, just for intuition, that two parties hold additive shares of the secret key and nonce:

$$a = a_1 + a_2,\qquad r = r_1 + r_2$$

Substitute those into the Schnorr equation:

$$S = r + c a = (r_1 + r_2) + c(a_1 + a_2)$$

Rearrange the terms:

$$S = (r_1 + c a_1) + (r_2 + c a_2)$$

Now each party can compute a local partial signature:

$$S_1 = r_1 + c a_1,\qquad S_2 = r_2 + c a_2$$

And the final signature is just:

$$S = S_1 + S_2$$

That is the clean part. The algebra closes under addition instead of forcing the parties into awkward secure multiplication machinery. Real threshold Schnorr systems usually use more general linear sharing schemes than this toy two-party split, but the reason they work so cleanly is the same.

At a high level, threshold Schnorr looks like this:

1. each signer contributes a nonce point
2. the coordinator aggregates them
3. each signer computes a local partial signature
4. the partial signatures are summed into one final signature

Schnorr systems are not trivial. Real systems still need good protocol design, malicious-party protection, careful nonce handling, and solid implementations. But the core signing equation is no longer actively hostile to distribution.

This is why threshold Schnorr systems such as FROST feel much lighter than threshold ECDSA lineages such as Lindell 2017, GG20, or CGGMP.

Same custody goal. Different cryptographic ergonomics.

## EdDSA in Brief

EdDSA is best understood as a concrete Schnorr-family design, not as a completely separate species.

For MPC, that is good news: its core signing structure is still Schnorr-like, so it inherits the additive property that makes threshold signing much cleaner than under ECDSA.

The main wrinkle is nonce handling. EdDSA standardizes deterministic nonce derivation, which is great on a single device but does not transfer directly to a setting where the full secret is never reconstructed. So threshold EdDSA still needs careful nonce coordination, whether through commit-and-reveal or FROST-style preprocessing and binding. It still belongs on the more cooperative side of the line.

## Why This Changes Wallet Engineering

To anyone not building custody infrastructure, this can sound like a niche technical footnote. In practice, it leaks all the way up into product and operations.

This is part of the reason secure wallet infrastructure is harder, slower, and more expensive than it first appears. It is why custody vendors end up talking about presigning, extra rounds, proof systems, and protocol families instead of just "splitting the key." Some systems can be made to feel relatively light. Others inherit heavier ceremony, more moving parts, and more places for implementation mistakes to hide.

Underneath a lot of crypto UX sits a stubborn constraint: the chain only accepts the signature scheme it accepts. If that scheme is ECDSA, distributed signing is possible, but it comes with real complexity. If that scheme is Schnorr-style, the math cooperates much more readily.

## The Short Version

> MPC wallets are not hard merely because multiple parties are involved. They get harder or easier depending on whether the underlying signature equation cooperates with distribution.

- `ECDSA`: the algebra fights you
- `Schnorr`: the algebra helps you
- `EdDSA`: the algebra helps you, but nonce handling still needs care

That is why two products can both market themselves as "MPC wallets" while hiding very different levels of cryptographic and operational complexity underneath.

## Further Reading

- [Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
- [CGGMP21 / CGGMP24](https://eprint.iacr.org/2021/060)
- [FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)
- [IETF FROST draft](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)

If you want the full derivations, protocol sketches, and implementation notes, I have a longer companion research note that expands the math behind this piece.
