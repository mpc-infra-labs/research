# If No One Has the Key, Who Signs?

*MPC wallets promise distributed trust. Blockchains still demand an ordinary signature. The gap between those two facts is why threshold ECDSA feels so much heavier than Schnorr.*

When a wallet vendor says it uses MPC, what is it actually promising?

Usually the promise is that no single machine, employee, or compromised server can move funds alone. That sounds like a straightforward engineering problem: split the key into pieces, distribute them across different systems, require a quorum, and call it a day.

In practice, though, "MPC wallet" can describe very different architectures, trust assumptions, and failure modes. The label tells you much less than it seems to.

But a blockchain does not accept "distributed trust" as an output. It accepts a normal signature. If a chain expects an ECDSA signature, your distributed system still has to produce a perfectly ordinary ECDSA signature without ever reconstructing the private key in one place.

That is where MPC wallets stop being a storage story and become a cryptographic engineering problem.

This matters even if you never plan to build a custody company. It helps explain why crypto wallet infrastructure is expensive, why some security architectures are much harder to audit or even evaluate from the outside than they first appear, and why the same phrase, "MPC wallet," can hide very different amounts of cryptographic machinery underneath.

It also explains why the underlying chain matters so much. Ethereum still forces one set of tradeoffs through ECDSA, and much of the Bitcoin stack in practice does too. Schnorr-based systems make a different set of tradeoffs available. Those differences are not branding. They come from the signature math.

That is the core claim of this article:

> Threshold signing is not equally hard across signature schemes. ECDSA is painful because its signing equation is non-linear. Schnorr-style signatures are cleaner because their signing equation stays additive.

If you understand that one distinction, a lot of the wallet and custody landscape starts looking much less arbitrary.

## What MPC Is Actually Trying To Do

The simplest way to think about wallet security is still the wrong one.

Most people imagine key management as a storage problem: where is the private key kept, and how do you keep someone from stealing it?

For serious custody systems, the problem is better described as an authorization problem. A private key can control an enormous amount of value. If one device, one admin, or one fallback process can produce a valid signature alone, then the real security boundary is just that weakest point.

Threshold signing changes that boundary.

It changes where trust sits. It does not eliminate governance failures, client compromise, insider abuse, or bad operational policy.

From the chain's point of view, there is still one public key and one valid signature. But behind the scenes, signing power is distributed across multiple parties, and no one party ever holds the complete signing key outright.

That is different from ordinary multisig.

Multisig changes what the chain sees. It exposes multiple keys, explicit script structure, or explicit policy logic on-chain. Threshold signing tries to preserve the external appearance of an ordinary wallet while distributing trust off-chain.

That flexibility can be an operational advantage. It can also make the real trust boundary harder for outsiders to inspect.

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

- `ECDSA`, because Ethereum and much of today's Bitcoin infrastructure still run on it
- `Schnorr`, because it is the cleaner algebraic alternative
- `EdDSA`, because it is the most important deployed Schnorr-family construction, especially through Ed25519

The useful mental model is not that these are three equally distant options. It is this:

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

If you do not care about the notation, that is fine. The only term worth staring at is the ECDSA inverse:

$$k^{-1}$$

That term is the biggest algebraic reason threshold ECDSA gets unpleasant.

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

And those algebraic costs do not stay in the math layer. They show up as more rounds, more presigning, more proof machinery, more implementation surface, and more ways for a production system to become slow, brittle, or hard to audit.

This is why people who work on threshold ECDSA sound slightly haunted. They are not being dramatic. They are describing a protocol family that fights the signature equation the whole way through.

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

## A Brief Note on EdDSA

EdDSA is best understood as a concrete Schnorr-family design, not as a completely separate species.

That is good news for MPC. Its core signing structure is still Schnorr-like, so it inherits the additive property that makes threshold signing much cleaner than under ECDSA.

The main wrinkle is nonce handling. EdDSA standardizes deterministic nonce derivation, which is great on a single device but does not transfer directly to a setting where the full secret is never reconstructed. So threshold EdDSA still needs careful nonce coordination, whether through commit-and-reveal or FROST-style preprocessing and binding. It still belongs on the more cooperative side of the line.

## Why This Is Worth Caring About

If you are not building custody infrastructure yourself, this can still sound like a niche technical footnote. It is not.

This is part of the reason secure wallet infrastructure is harder, slower, and more expensive than it first appears. It is why custody vendors talk about presigning, extra rounds, proof systems, and protocol families instead of just "splitting the key." Some systems can be made to feel relatively light. Others inherit heavier ceremony, more moving parts, and more places for implementation mistakes to hide.

Underneath a lot of crypto UX, there is still a very old and very stubborn fact: the chain only accepts the signature scheme it accepts. If that scheme is ECDSA, distributed signing is possible, but it comes with real complexity. If that scheme is Schnorr-style, the math cooperates much more readily.

## The Short Version

If you only keep one sentence from this article, keep this one:

> MPC wallets are not hard merely because multiple parties are involved. They get harder or easier depending on whether the underlying signature equation cooperates with distribution.

Or even shorter:

- `ECDSA`: the algebra fights you
- `Schnorr`: the algebra helps you
- `EdDSA`: the algebra helps you, but nonce handling still needs care

That is the simplest explanation I know for why two products can both market themselves as "MPC wallets" while hiding very different levels of cryptographic and operational complexity underneath.

## Further Reading

- [Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
- [CGGMP21 / CGGMP24](https://eprint.iacr.org/2021/060)
- [FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)
- [IETF FROST draft](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)

If you want the full derivations, protocol sketches, and implementation notes, I have a longer companion research note that expands the math behind this piece.
