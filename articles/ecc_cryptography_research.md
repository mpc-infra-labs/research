# Why ECDSA Makes MPC Wallets So Painful

Most people first hear "MPC wallet" and picture a vague security upgrade: split the key across several machines, require a quorum, and keep any one compromise from becoming catastrophic.

That intuition is directionally right. It is also incomplete.

The hard part is not just that multiple parties are involved. The hard part is that blockchains still expect an ordinary signature from an ordinary signature scheme. If the chain expects an ECDSA signature, your distributed system has to jointly produce a valid ECDSA signature without ever reconstructing the private key in one place.

That requirement sounds reasonable until you look at the algebra.

For some signature schemes, threshold signing is almost annoyingly elegant. For others, it becomes a pile of auxiliary cryptography: homomorphic encryption, zero-knowledge proofs, extra preprocessing, more communication rounds, more implementation risk.

ECDSA is in the second category.

This is the core point of the article:

> Threshold signing is not equally hard across signature schemes. ECDSA is painful because its signing equation is non-linear. Schnorr-style signatures are cleaner because their signing equation stays additive.

If you understand that one distinction, a lot of the MPC landscape starts making sense.

## The Real Custody Problem

Institutional crypto custody is not fundamentally a storage problem. It is an authorization problem.

A private key can control an enormous amount of value. If one laptop, one HSM, one operator, or one recovery flow can produce a valid signature alone, then your entire security model collapses to that single point of failure.

That is why threshold signing exists.

From the blockchain's point of view, nothing exotic happens. The chain still sees one public key and one valid signature. But behind the scenes, signing power is distributed across multiple parties, and no one party ever holds the complete signing key outright.

That is different from ordinary multisig.

Multisig changes what the chain sees. Threshold signing does not. Threshold signing keeps the external interface looking like a normal wallet while moving the trust distribution off-chain.

This is extremely attractive for exchanges, custodians, treasury systems, and any infrastructure stack that wants:

- quorum-based control instead of single-device control
- survivability if one signer or one machine disappears
- reduced blast radius if one component is compromised

So far, so good.

But then the signature scheme starts to matter.

## The One Bit of ECC You Actually Need

We can skip almost all of elliptic-curve cryptography and keep only the part that matters here.

- A private key is a scalar, usually written as `d`.
- A public key is a curve point derived from it: `Q = d * G`.
- Signature schemes differ in how they combine the private key, a message hash, and a nonce.

For threshold signing, that last bullet is the whole story.

If the signature equation is additive, parties can usually hold additive shares and combine local work cleanly.

If the signature equation forces multiplication across secrets held by different parties, life gets worse quickly.

## Three Signature Families, Three Very Different Ergonomics

For custody infrastructure, three schemes matter most:

- `ECDSA`, because Bitcoin and Ethereum still depend on it
- `Schnorr`, because it is the clean algebraic alternative and powers modern Bitcoin threshold constructions
- `EdDSA`, because it is the most important deployed Schnorr-family design, especially via Ed25519

The useful mental model is not "three unrelated schemes."

It is this:

- `ECDSA` is the awkward one for MPC
- `Schnorr` is the clean one
- `EdDSA` mostly inherits Schnorr's clean structure, but adds a nonce-coordination wrinkle

Here are the core signing equations:

ECDSA:

$$s = k^{-1}(z + r d) \pmod{n}$$

Schnorr:

$$S = r + c a$$

EdDSA:

$$S = r + c a$$

That single visual contrast explains far more than it first appears to.

In ECDSA, there is an inversion term, `k^{-1}`.

In Schnorr-style signatures, the equation stays additive from end to end.

That is the fork in the road.

## Why ECDSA Threshold Signing Gets Ugly

Suppose we want multiple parties to sign without ever reconstructing the private key on one machine.

At first glance, that sounds like a straightforward secret-sharing problem. Split the key, have each party do some local math, combine the results, and return a normal signature.

That is not what happens with ECDSA.

ECDSA signing is:

$$s = k^{-1}(z + r d) \pmod{n}$$

The private key `d` is secret. The nonce `k` is also secret. In a threshold setting, different parties hold shares of both.

Now expand the intuition a bit. Even if you distribute the key material, the protocol still has to evaluate a structure that mixes:

- the inverse of a secret nonce
- the message hash
- the product of the public nonce component `r` and the secret signing key

The problem is not only secrecy. The problem is that the computation is not naturally share-friendly.

In practical threshold ECDSA protocols, this eventually becomes a secure multiplication problem: parties need to obtain the effect of multiplying secrets they do not want to reveal to one another.

That is where the heavy machinery enters.

Classical threshold ECDSA designs use techniques such as:

- Paillier-based additive homomorphic encryption
- MtA, or multiplicative-to-additive conversion
- zero-knowledge proofs to ensure everyone followed the protocol honestly

All of that is there for a reason. It is not ornamental complexity. It is compensating for the fact that ECDSA's algebra does not cooperate with distributed signing.

This is why threshold ECDSA libraries and papers often feel much heavier than outsiders expect. The system is fighting the signature equation the whole time.

## Why Schnorr Keeps Showing Up as the "Cleaner" Alternative

Now compare that with Schnorr:

$$S = r + c a$$

Everything important is additive.

If parties hold additive shares of the private key and additive shares of the nonce, they can compute partial signatures locally and add them together. The algebra closes cleanly.

At a high level, threshold Schnorr feels like:

1. each signer contributes a nonce point
2. the coordinator aggregates them into a joint nonce
3. each signer computes a local partial signature
4. the partial signatures are summed into the final signature

That is why threshold Schnorr protocols like FROST are so much cleaner than threshold ECDSA lineages such as Lindell 2017, GG20, or CGGMP.

The custody problem is the same. The cryptographic ergonomics are not.

## Where EdDSA Fits

EdDSA is best understood as a concrete Schnorr-family design, not as a third completely separate species.

Its core signing structure is still Schnorr-like, which is good news for MPC. It inherits the nice additive property that makes threshold signing much easier than under ECDSA.

But EdDSA also standardizes deterministic nonce derivation.

In ordinary single-device Ed25519 signing, that is a feature. You do not depend on a fresh random nonce generator every time, which removes an entire category of catastrophic implementation failures.

In MPC, though, deterministic nonce generation becomes awkward.

Why? Because the whole point of threshold custody is that the complete secret material is never reconstructed in one place. So the protocol cannot simply evaluate the usual single-device nonce derivation procedure as written.

That is why threshold EdDSA systems often replace single-device determinism with coordinated randomness and session binding:

- simpler constructions use commit-and-reveal
- faster constructions use FROST-style nonce preprocessing

So EdDSA is still much friendlier than ECDSA for threshold signing. It just is not completely free of protocol design work.

## The Practical Bottom Line

If you only keep one sentence from this article, keep this one:

> MPC wallets are not hard merely because multiple parties are involved. They are hard or easy depending on whether the underlying signature equation cooperates with distribution.

That gives you a much better map of the space:

- if a chain forces you to use `ECDSA`, expect heavier protocols and more implementation complexity
- if you can use `Schnorr`, threshold signing gets dramatically cleaner
- if you use `EdDSA`, you mostly get Schnorr's algebraic benefits, but you still need careful nonce coordination

This is why the modern protocol landscape looks the way it does.

Threshold ECDSA systems tend to accumulate MtA, homomorphic encryption, proofs, presigning flows, and a larger implementation surface. Threshold Schnorr systems tend to look much more natural. Same custody goal. Very different mathematical starting point.

## A Short Mental Model

You can compress the whole article into three lines:

- `ECDSA`: the algebra fights you
- `Schnorr`: the algebra helps you
- `EdDSA`: the algebra helps you, but nonce handling still needs care

That is the simplest explanation I know for why MPC wallet infrastructure can feel either surprisingly elegant or surprisingly painful.

## Further Reading

- [Lindell 2017 — Fast Secure Two-Party ECDSA Signing](https://eprint.iacr.org/2017/552)
- [CGGMP21 / CGGMP24](https://eprint.iacr.org/2021/060)
- [FROST — Flexible Round-Optimized Schnorr Threshold Signatures](https://eprint.iacr.org/2020/852)
- [IETF FROST draft](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/)

If you want the full derivations, protocol sketches, and implementation notes, read the companion research note in [research_notes/ecc_cryptography_and_mpc_research_note.md](/Users/wenyiyu/wenyi/mpc-labs/mpc-research/research_notes/ecc_cryptography_and_mpc_research_note.md).
