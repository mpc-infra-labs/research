# Why Secure Crypto Wallets Are Harder Than They Look

*The hard part is not splitting a private key. The hard part is producing an ordinary blockchain signature without ever reconstructing that key anywhere.*

When a crypto exchange, custodian, or wallet provider says it uses MPC, the promise sounds simple enough: no single machine, employee, or compromised server should be able to move funds alone.

That sounds like a straightforward engineering problem. Split the key into pieces. Put the pieces on different machines. Require a quorum. Done.

That is not how it works.

The blockchain does not care how sophisticated your internal security architecture is. It still expects an ordinary signature from an ordinary signature scheme. If the chain expects an ECDSA signature, your distributed system has to jointly produce a valid ECDSA signature without ever reconstructing the private key in one place.

That is where things stop being simple.

This matters even if you never plan to build a custody company. It helps explain why crypto wallet infrastructure is expensive, why some security architectures are much harder to audit than they first appear, and why the same phrase, "MPC wallet," can hide very different amounts of cryptographic machinery underneath.

It also explains why the underlying chain matters so much. Bitcoin and Ethereum force one set of tradeoffs. Schnorr-based systems make a different set of tradeoffs available. Those differences are not branding. They come from the signature math.

That is the core claim of this article:

> Threshold signing is not equally hard across signature schemes. ECDSA is painful because its signing equation is non-linear. Schnorr-style signatures are cleaner because their signing equation stays additive.

If you understand that one distinction, a lot of the wallet and custody landscape starts looking much less arbitrary.

## What MPC Is Actually Trying To Do

The simplest way to think about wallet security is still the wrong one.

Most people imagine key management as a storage problem: where is the private key kept, and how do you keep someone from stealing it?

For serious custody systems, the problem is better described as an authorization problem. A private key can control an enormous amount of value. If one device, one admin, or one fallback process can produce a valid signature alone, then the real security boundary is just that weakest point.

Threshold signing changes that boundary.

From the chain's point of view, there is still one public key and one valid signature. But behind the scenes, signing power is distributed across multiple parties, and no one party ever holds the complete signing key outright.

That is different from ordinary multisig.

Multisig changes what the chain sees. It exposes multiple keys, explicit script structure, or explicit policy logic on-chain. Threshold signing tries to preserve the external appearance of an ordinary wallet while distributing trust off-chain.

Why does that matter to a normal reader?

Because this is the infrastructure that sits underneath exchanges, custodians, treasury systems, and any product that wants stronger security without turning every transaction into a visibly exotic on-chain construction. If you have ever wondered why some wallet systems are more expensive, slower to ship, harder to audit, or more operationally fragile than others, this is part of the answer.

## The One Technical Fact You Need

We can skip most of elliptic-curve cryptography and keep only the part that matters here.

- A private key is a secret number.
- A public key is derived from it.
- A signature scheme specifies how a private key, a message, and a nonce combine to produce a valid signature.

For threshold signing, that last point is the whole story.

If the signature equation is additive, parties can often hold additive shares and combine their local work cleanly.

If the signature equation forces multiplication across secrets held by different parties, life gets worse quickly.

That is the difference between "this is an elegant protocol" and "this is a large cryptographic engineering project."

## The Signature Scheme Choice Is Not Cosmetic

For custody infrastructure, three schemes matter most:

- `ECDSA`, because Bitcoin and Ethereum still depend on it
- `Schnorr`, because it is the cleaner algebraic alternative
- `EdDSA`, because it is the most important deployed Schnorr-family construction, especially through Ed25519

The useful mental model is not that these are three equally different signature schemes.

The useful mental model is this:

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

If you do not care about the notation, that is fine. The only part worth staring at is the ECDSA inverse:

$$k^{-1}$$

That one term is the reason threshold ECDSA gets unpleasant.

In Schnorr-style signatures, the algebra stays additive from end to end.

That is the fork in the road.

## Why ECDSA Makes MPC So Heavy

At first glance, threshold signing sounds like a secret-sharing problem.

Take a private key. Split it across several parties. Have each party do its part. Combine the results. Return one valid signature.

That intuition is close enough to be dangerous.

ECDSA does not only require secret values. It requires a computation over those secret values that does not naturally decompose across parties.

The signing equation is:

$$s = k^{-1}(z + r d) \pmod{n}$$

The private key `d` is secret. The nonce `k` is secret. In a threshold setting, different parties hold different shares of both.

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

This is why people who work on threshold ECDSA sound slightly haunted. They are not being dramatic. They are describing a protocol family that is fighting the signature equation the whole time.

## Why Schnorr Feels Much Cleaner

Now compare that with Schnorr:

$$S = r + c a$$

Everything important is additive.

If parties hold additive shares of the private key and additive shares of the nonce, they can compute partial signatures locally and add them together. The algebra closes cleanly.

At a high level, threshold Schnorr looks like this:

1. each signer contributes a nonce point
2. the coordinator aggregates them
3. each signer computes a local partial signature
4. the partial signatures are summed into one final signature

That does not make Schnorr systems trivial. Real systems still need good protocol design, malicious-party protection, careful nonce handling, and solid implementations. But the core signing equation is no longer actively hostile to distribution.

This is why threshold Schnorr systems such as FROST feel much lighter than threshold ECDSA lineages such as Lindell 2017, GG20, or CGGMP.

Same custody goal. Different cryptographic ergonomics.

## Where EdDSA Fits

EdDSA is best understood as a concrete Schnorr-family design, not as a completely separate species.

That is good news for MPC. Its core signing structure is still Schnorr-like, so it inherits the nice additive property that makes threshold signing much cleaner than under ECDSA.

But EdDSA also standardizes deterministic nonce derivation.

On a single device, that is a feature. It removes an entire class of catastrophic randomness failures.

In MPC, it becomes awkward. The whole point of threshold signing is that the complete secret material is never reconstructed in one place. So the standard single-device nonce derivation procedure does not carry over directly.

That is why threshold EdDSA systems still need careful nonce coordination:

- simpler constructions use commit-and-reveal
- faster constructions use FROST-style nonce preprocessing and binding

So EdDSA is much friendlier than ECDSA for threshold signing. It is just not free of protocol design work.

## Why This Is Worth Caring About

If you are not building custody infrastructure yourself, this can still sound like a niche technical footnote. It is not.

This is part of the reason secure wallet infrastructure is harder, slower, and more expensive than it first appears. It is part of the reason custody vendors talk about presigning, rounds, proof systems, and protocol families instead of just "splitting the key." It is part of the reason chain choice leaks upward into product and security design.

Underneath a lot of crypto UX, there is still a very old and very stubborn fact: the chain only accepts the signature scheme it accepts.

If that scheme is ECDSA, distributed signing is possible, but it comes with real complexity.

If that scheme is Schnorr-style, the math cooperates much more readily.

That is not an implementation detail. It is one of the reasons wallet infrastructure across chains ends up feeling so uneven.

## The Short Version

If you only keep one sentence from this article, keep this one:

> MPC wallets are not hard merely because multiple parties are involved. They are hard or easy depending on whether the underlying signature equation cooperates with distribution.

Or even shorter:

- `ECDSA`: the algebra fights you
- `Schnorr`: the algebra helps you
- `EdDSA`: the algebra helps you, but nonce handling still needs care

That is the simplest explanation I know for why some cryptographic wallet systems look surprisingly elegant and others look like they are held together by advanced math and careful prayer.

## Further Reading

- [Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
- [CGGMP21 / CGGMP24](https://eprint.iacr.org/2021/060)
- [FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)
- [IETF FROST draft](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)

If you want the full derivations, protocol sketches, and implementation notes, read the companion research note in [research_notes/ecc_cryptography_and_mpc_research_note.md](/Users/wenyiyu/wenyi/mpc-labs/mpc-research/research_notes/ecc_cryptography_and_mpc_research_note.md).
