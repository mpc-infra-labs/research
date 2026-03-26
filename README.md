# mpc-research

Research notes, implementation notes, and article drafts for threshold signing infrastructure.

This repository documents the cryptographic and systems foundations behind the work in `mpc-infra-labs`.

It has three goals:

1. explain the base knowledge required to understand threshold signing systems
2. record implementation-level insights discovered while building real components
3. publish structured articles that connect protocol theory to working code

## Sections

- `articles/`
  - publishable research articles and long-form technical writing

- `experiments/`
  - experimental code and proof-of-concepts demonstrating cryptographic primitives

## Main Code Repositories

- `chain-processor-evm`
- `coordinator`
- `cosigner`
- `threshold-signing-demo`

## Article Series

- `00-series-intro.md`
- `01-ecc-cryptography-and-mpc-foundations.md`
- `02-why-ecdsa-mpc-is-hard.md`
- `03-from-threshold-signing-to-an-evm-transaction.md`
- `04-bridging-cggmp24-and-synedrion.md`
- `05-shamir-vs-additive-shares.md`
- `06-hd-wallets-bip32-and-threshold-signing.md`
- `07-running-round-based-mpc-over-http.md`
- `08-bitcoin-script-types-through-threshold-signing.md`

## Philosophy

The purpose of this repo is not to collect disconnected notes.

It is to build a coherent research trail for threshold signing infrastructure:
from ECC foundations, to MPC protocol tradeoffs, to implementation decisions, to transaction execution systems.
