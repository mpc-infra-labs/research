# Securing Client-Side Shares with Mobile Secure Enclaves

This article focuses on the unique challenges of MPC when one of the participating shares is held by an end-user on a mobile device. This is a common architecture for Web3 wallets that offer "seedless" recovery and social login.

## 1. The Mobile Hardware Security Landscape

- **iOS Secure Enclave (SEP)**: Architecture and capabilities.
- **Android TrustZone & StrongBox**: How it compares to the iOS equivalent.
- **Passkeys / WebAuthn**: How FIDO2 standards leverage this underlying hardware for phishing-resistant authentication.

## 2. The Core Challenge: MPC Protocols Cannot Run in Mobile Enclaves

- Explaining the limitations of the TEE environments on mobile devices (restricted instruction sets, no complex networking, etc.).
- Why a complex, round-based protocol like CGGMP24 cannot be executed directly inside the Secure Enclave.

## 3. Common Architectural Patterns

- **Pattern 1: Encrypting the Share**: The mobile device's hardware key is used to encrypt the user's MPC share. The encrypted share can be stored on the device or in a cloud backup (e.g., iCloud).
- **Pattern 2: Authentication as Authorization**: The user authenticates with their device biometrics (FaceID/TouchID), which unlocks a hardware-backed key. This key signs a request authorizing a server-side node to use its corresponding share in an MPC ceremony.
- **Case Studies**: Analyzing the architectures of services like Web3Auth or Coinbase WaaS.