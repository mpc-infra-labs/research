# Hardware Security in MPC: TEEs, HSMs, and Enclaves

This article explores the critical role of hardware security in building production-grade MPC custody systems. While MPC distributes trust at the cryptographic layer, each participating node remains a potential point of failure at the infrastructure layer. Hardware security provides the necessary isolation to protect key shares even on a compromised host.

## 1. The Threat Model: Why Software-Only MPC is Insufficient

- A compromised host (root access)
- Memory dumping attacks
- Malicious insiders / administrators

## 2. TEEs: The Institutional Standard (Intel SGX & AWS Nitro Enclaves)

- **Core Concepts**: How do Trusted Execution Environments work?
- **Case Study: Fireblocks and Intel SGX**: Analyzing the architecture of a leading custodian.
- **Deep Dive: AWS Nitro Enclaves**: Why Nitro Enclaves are a popular choice for modern cloud-native MPC deployments.
- **Remote Attestation**: The mechanism for cryptographically proving that code is running inside a genuine TEE.

## 3. Hybrid Models: Bridging Traditional Custody and MPC

- **HSM-backed MPC**: Using Hardware Security Modules (HSMs) to protect key shares at rest.
- **Cloud KMS Integration**: A more flexible approach using cloud provider services like AWS KMS or Google Cloud KMS to encrypt and decrypt shares just-in-time.
- **Trade-offs**: Comparing the security, cost, and agility of HSMs vs. Cloud KMS for this purpose.

## 4. The Future: Hardware Wallets as MPC Participants?

- Analyzing the technical limitations of current hardware wallets (Ledger, Trezor) for participating in interactive MPC protocols.
- Speculating on future firmware or hardware designs that could enable this.