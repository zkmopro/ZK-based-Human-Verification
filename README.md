# ZK-based Proof of Personhood for Online Forum

A proof-of-concept for ZK-based proof of personhood on online forum platforms — specifically PTT — using [OpenAC / zkID](https://github.com/privacy-ethereum/zkID/blob/main/paper/zkID.pdf) and Taiwan's mobile citizenship certificate (MOICA).

> For flow diagrams and architecture decisions, see [ARCHITECTURE.md](./ARCHITECTURE.md).
> For the formal ZK proof-of-personhood specification (circuit design, public inputs, nullifier scheme, and security properties), see the [spec](https://github.com/privacy-ethereum/zkID/tree/main/specs/2-zk-proof-of-personhood).

---

## Repository Structure

```
.
├── zkid/                                                      # [REUSABLE] ZK circuits and proof generation core (Circom + Spartan2 / Hyrax)
├── go-zkid-verifier/                                          # [REUSABLE CORE / PTT-INTEGRATED] Go server — challenge issuance, proof verification, nullifier deduplication
├── moica-revocation-smt/                                      # [REUSABLE] Revocation SMT pipeline — MOICA CRL → on-chain SMT root (Arbitrum Sepolia)
├── openac-rsa-x509-swift/                                     # [REUSABLE] iOS Swift bindings for RSA x509 cert on-device proof generation
├── openac-rsa-x509-kotlin/                                    # [REUSABLE] Android Kotlin bindings for RSA x509 cert on-device proof generation
├── openac-rsa-x509-js/                                        # [REUSABLE] JavaScript/Node.js SDK for RSA x509 cert proof generation (browser + Node.js)
├── openac-taiwan-citizen-digital-certificate-ios-example/     # [EXAMPLE] iOS sample app — full OpenAC ZK pipeline end-to-end
├── openac-taiwan-citizen-digital-certificate-android-example/ # [EXAMPLE] Android sample app — full OpenAC ZK pipeline end-to-end
├── openac-taiwan-citizen-digital-certificate-web-example/     # [EXAMPLE] Web sample app — HiPKI smart card, in-browser ZK proving
└── assets/                                                    # Flow diagrams
```

---

## Component Map

The table below labels each functional piece, its submodule, and whether it is a **reusable OpenAC/zkID building block** or a **PTT-specific integration concern**.

| Component | Where it lives | Reusable or PTT-specific |
|-----------|---------------|--------------------------|
| **Proof generation** (cert-chain + user-sig circuits, Circom + Spartan2/Hyrax) | `zkid/wallet-unit-poc/` | Reusable — any OpenAC relying party |
| **Proof generation — iOS** (Swift FFI + on-device proving, x509 cert circuits) | `openac-rsa-x509-swift/` | Reusable for any x509 certificate-based identity scheme using the same circuits |
| **Proof generation — Android** (Kotlin/JNI FFI + on-device proving, x509 cert circuits) | `openac-rsa-x509-kotlin/` | Reusable for any x509 certificate-based identity scheme using the same circuits |
| **Proof generation — Web** (in-browser WASM + HiPKI smart card, x509 cert circuits) | `openac-rsa-x509-js/` + `openac-taiwan-citizen-digital-certificate-web-example/` | Reusable SDK; example app is Taiwan citizen digital certificate specific |
| **Proof verification** (Rust FFI → Go, Spartan2 verifier) | `go-zkid-verifier/verifier/` | Reusable — exposes `linkverify.Service` for any backend |
| **Nullifier generation** (derived inside `user_sig` circuit from `APP_ID` + cardholder key) | `zkid/wallet-unit-poc/circom/` | Reusable design; `APP_ID` is per–relying party |
| **Challenge / nonce handling** (per-session 254-bit field element, 5-min TTL, replay protection) | `go-zkid-verifier/challenge/` + `httpapi/challenge.go` | Reusable pattern; `APP_ID` is PTT-specific |
| **Revocation SMT — server** (CRL fetch, Poseidon SMT, REST/gRPC proofs, on-chain root posting) | `moica-revocation-smt/server/` | Reusable — any MOICA-issuer integration |
| **Revocation SMT — client** (offline snapshot + proof generation, iOS/Android/WASM) | `moica-revocation-smt/` + `OpenACSwift/` | Reusable — any mobile or web client using MOICA |
| **Revocation SMT — on-chain root** (`SMTRootStorage.sol`, Arbitrum Sepolia) | `moica-revocation-smt/onchain-contract/` | Reusable — any verifier can read the root |
| **Revocation root cache** (on-chain primary → GitHub-release fallback, background refresh) | `go-zkid-verifier/smtroot/` | Reusable — decoupled provider interface |
| **Issuer cert trust** (MOICA-G2/G3 moduli, embedded + HTTPS refresh, GRCA chain-validate) | `go-zkid-verifier/issuercert/` | Reusable — applicable to any MOICA verifier |
| **PTT backend integration** (HTTP/gRPC server, `link-verify` endpoint, SQLite deduplication) | `go-zkid-verifier/httpapi/` + `grpc/` + `store/` | PTT-specific deployment; core logic is reusable |
| **Mobile app integration** (online forum app fetches MOICA cert, wraps VC, calls Swift/Kotlin FFI) | `openac-rsa-x509-swift/` + `openac-rsa-x509-kotlin/` | Reusable for any x509 cert-based identity scheme; PTT app shell is PTT-specific |
| **iOS example app** (full end-to-end demo: TW FidO auth → circuit download → ZK prove → link-verify) | `openac-taiwan-citizen-digital-certificate-ios-example/` | Reusable reference implementation for any OpenAC iOS integration |
| **Android example app** (full end-to-end demo: TW FidO auth → circuit download → ZK prove → link-verify) | `openac-taiwan-citizen-digital-certificate-android-example/` | Reusable reference implementation for any OpenAC Android integration |
| **Web example app** (HiPKI smart card → in-browser ZK prove → link-verify) | `openac-taiwan-citizen-digital-certificate-web-example/` | Reusable reference implementation for any OpenAC web integration |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   REUSABLE OpenAC / zkID LAYER                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              zkid  (ZK circuit core)                    │   │
│  │  cert_chain_rs4096   ──┐  Circom circuits               │   │
│  │  user_sig_rs2048     ──┤  Spartan2 + Hyrax              │   │
│  │                         └→ nullifier derived here       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────┐   ┌────────────────────────────────┐   │
│  │  openac-rsa-x509-    │   │  openac-rsa-x509-kotlin        │   │
│  │  swift               │   │  Android JNI + prove           │   │
│  │  iOS FFI + prove     │   │  + SMT client                  │   │
│  │  + SMT client        │   └────────────────────────────────┘   │
│  └──────────────────────┘                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openac-rsa-x509-js  (browser WASM + Node.js SDK)        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              moica-revocation-smt                       │   │
│  │  CRL fetch → Poseidon SMT → REST/gRPC proof API        │   │
│  │  on-chain root (SMTRootStorage, Arbitrum Sepolia)       │   │
│  │  binary snapshots for WASM / mobile offline use         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │    go-zkid-verifier  (reusable verifier core)           │   │
│  │  linkverify.Service  (transport-agnostic orchestrator)  │   │
│  │  challenge issuance  │  nullifier dedup  │  SMT check   │   │
│  │  issuer-cert check   │  app_id check     │  FFI verify  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                    PTT-SPECIFIC INTEGRATION
                              │
              ┌───────────────┴────────────────┐
              │   go-zkid-verifier             │
              │   HTTP :8080  /challenge        │
              │              /link-verify       │
              │   gRPC :9090  ZkIDVerifier      │
              │   SQLite  challenges + records  │
              │   APP_ID = PTT relying-party ID │
              └────────────────────────────────┘
```

**Boundary summary:**
- Everything above the "PTT-SPECIFIC INTEGRATION" line is reusable by any relying party implementing OpenAC, with one caveat: `openac-rsa-x509-swift`, `openac-rsa-x509-kotlin`, and `openac-rsa-x509-js` are built for RSA x509 certificate-based identity circuits (`cert_chain_rs4096`, `user_sig_rs2048`). They are drop-in reusable for any x509 cert scheme; for other circuit designs the bindings must be regenerated via mopro.
- The PTT-specific surface is narrow: the `APP_ID` env var, the SQLite deduplication store, and the HTTP/gRPC transport wiring. Swapping these out adapts the verifier to a different service.

---

## Submodules

| Submodule | Role | Upstream | Pinned commit |
|-----------|------|----------|---------------|
| [zkid](https://github.com/privacy-ethereum/zkID/tree/RSA-X.509-Cert) | ZK circuits (`cert_chain_rs4096`, `user_sig_rs2048`), Spartan2/Hyrax prover, Circom witness calculator | [PSE zkID](https://github.com/privacy-ethereum/zkID) | [`8e599a7`](https://github.com/privacy-ethereum/zkID/commit/8e599a73f2a43d8c2ecc4b62f32614efd220cedd) |
| [go-zkid-verifier](https://github.com/privacy-ethereum/go-zkid-verifier) | Go backend: challenge/nonce, ZK proof verification (FFI → Rust), nullifier dedup, SMT root and issuer-cert trust checks | — | [`4bd1fdb`](https://github.com/privacy-ethereum/go-zkid-verifier/commit/4bd1fdb32af17ecae6b00917234cbe235011cc0f) |
| [moica-revocation-smt](https://github.com/privacy-ethereum/moica-revocation-smt) | MOICA CRL → Poseidon SMT → REST/gRPC proofs → on-chain root (Arbitrum Sepolia `0xc461…AFFA`); binary snapshots for offline client use | — | [`f23b82d`](https://github.com/privacy-ethereum/moica-revocation-smt/commit/f23b82d2f498eab157539cbe4ce5ecd4a003ab28) |
| [openac-rsa-x509-swift](https://github.com/privacy-ethereum/openac-rsa-x509-swift) | Swift package (iOS 16+): on-device proof generation for RSA x509 cert circuits (`proveCertChainRs4096`, `proveUserSigRs2048`), offline SMT revocation check, circuit-input preparation | [mopro](https://github.com/zkmopro/mopro) | [`cce06e1`](https://github.com/privacy-ethereum/openac-rsa-x509-swift/commit/cce06e1a3d81adde8f399faf49fd307cf49b3a27) |
| [openac-rsa-x509-kotlin](https://github.com/privacy-ethereum/openac-rsa-x509-kotlin) | Kotlin/Android library (JNI): on-device proof generation for RSA x509 cert circuits (`proveCertChainRs4096`, `proveUserSigRs2048`), circuit-input preparation | [mopro](https://github.com/zkmopro/mopro) | [`a701d4d`](https://github.com/privacy-ethereum/openac-rsa-x509-kotlin/commit/a701d4decc41855f1f6ef5aee5c485ca68334ba4) |
| [openac-rsa-x509-js](https://github.com/privacy-ethereum/openac-rsa-x509-js) | JavaScript/Node.js SDK: browser WASM (wasm-bindgen) + Node.js wrappers for RSA x509 cert proof generation (`prove`, `verify`, `load_pk`, `build_split_inputs`); `browser` export condition for Vite | — | [`62be9ed`](https://github.com/privacy-ethereum/openac-rsa-x509-js/commit/62be9ed20d18c465183191ba37d6ac93d3d467e8) |
| [openac-taiwan-citizen-digital-certificate-ios-example](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-ios-example) | iOS sample app: HiPKI / MOICA auth, circuit-key download, ZK proof generation, and `POST /link-verify` submission | [openac-rsa-x509-swift](https://github.com/privacy-ethereum/openac-rsa-x509-swift) | [`cbc9dc3`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-ios-example/commit/cbc9dc39612f7e15168468e425aec55b5d0609ab) |
| [openac-taiwan-citizen-digital-certificate-android-example](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-android-example) | Android sample app (Jetpack Compose): HiPKI / MOICA auth, circuit-key download, ZK proof generation, and `POST /link-verify` submission | [openac-rsa-x509-kotlin](https://github.com/privacy-ethereum/openac-rsa-x509-kotlin) | [`be79f2d`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-android-example/commit/be79f2d84009b786c912cd324e8272b5646c06d0) |
| [openac-taiwan-citizen-digital-certificate-web-example](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-web-example) | Web sample app: HiPKI smart card (自然人憑證) → in-browser ZK proving (rayon + SharedArrayBuffer) → `POST /link-verify` | [openac-rsa-x509-js](https://github.com/privacy-ethereum/openac-rsa-x509-js) | [`1c19817`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-web-example/commit/1c198174d62def75c95844889230bf204cca0a70) |

---

## Verification Flow (Mobile — PTT)

```
Mobile app
  1. Fetch MOICA certificate via TW government API
  2. Wrap raw cert data into JWT-format VC
  3. Call GET /challenge on go-zkid-verifier → receive {challenge, app_id}
  4. Generate ZK proof on-device (OpenACSwift / OpenACKotlin)
       - cert_chain_rs4096: proves cert was signed by MOICA-G2/G3 and is not revoked (SMT non-membership)
       - user_sig_rs2048: proves user signed app_id with the user key; derives nullifier
  5. POST /link-verify {cert_chain_proof, user_sig_proof}

go-zkid-verifier
  6. Verify both proofs via Rust FFI (Spartan2 / Hyrax)
  7. Check smt_root public input against moica-revocation-smt (on-chain, Arbitrum Sepolia)
  8. Check issuer_rsa_modulus against embedded MOICA-G2/G3 certs
  9. Check app_id matches configured PTT APP_ID
  10. Check challenge matches issued nonce (anti-replay)
  11. Check nullifier is not already recorded (one-time-per-credential)
  12. Return {verified: true, nullifier} → online forum server marks user as personhood-verified
```

---

## Upstream References

| Resource | Link |
|----------|------|
| OpenAC / zkID paper | [zkID.pdf](https://github.com/privacy-ethereum/zkID/blob/main/paper/zkID.pdf) |
| ZK proof-of-personhood spec (circuit design, public inputs, nullifier scheme, security properties) | [specs/2-zk-proof-of-personhood](https://github.com/privacy-ethereum/zkID/tree/main/specs/2-zk-proof-of-personhood) |
| PSE zkID team (upstream wallet-unit-poc) | [privacy-ethereum/zkID](https://github.com/privacy-ethereum/zkID/tree/main/wallet-unit-poc) |
| mopro (mobile ZK framework) | [zkmopro/mopro](https://github.com/zkmopro/mopro) |
| zkID 2026 roadmap | [Notion](https://pse-team.notion.site/zkID-2026-Roadmap-2fdd57e8dd7e80f48a37c24e9fbe09d6) |
| zkID benchmarks | [csp-benchmarks](https://github.com/privacy-ethereum/csp-benchmarks) |
| SMTRootStorage contract (Arbitrum Sepolia) | [`0xc461326eb6e46F10A276B0F14BFFf8b256A43FFA`](https://sepolia.arbiscan.io/address/0xc461326eb6e46F10A276B0F14BFFf8b256A43FFA) |

---

## Reuse Guide for Third Parties

To adapt this integration for a non-PTT relying party:

1. **Circuits** — use `zkid` unchanged; verifying keys are published on [zkID `RSA-X.509-Cert` releases](https://github.com/privacy-ethereum/zkID/releases/tag/RSA-X.509-Cert-latest).
2. **Bindings** — add `openac-rsa-x509-swift` (SPM), `openac-rsa-x509-kotlin` (JitPack), or `openac-rsa-x509-js` (npm) as dependencies if your identity scheme is x509 certificate-based; call `generateCertChainRs4096Input → setupKeys → prove* → linkVerify`. If your circuits differ from the x509 cert design, regenerate the bindings via mopro against your own circuit artifacts.
3. **Revocation** — consume the [moica-revocation-smt](https://github.com/privacy-ethereum/moica-revocation-smt) REST/gRPC proof API, or load a binary snapshot locally (WASM / iOS / Android).
4. **Backend verifier** — deploy `go-zkid-verifier` with `APP_ID` set to your relying-party identifier (31-char hex); wire `linkverify.Service` into your own transport or use the bundled HTTP/gRPC server.
5. **What to change** — only `APP_ID`, your session store backend, and any online-forum-specific user-DB update logic. The ZK circuit, mobile bindings, and SMT infrastructure are drop-in.

---

## Getting Started

### Clone

```bash
git clone --recurse-submodules https://github.com/privacy-ethereum/ZK-based-Proof-of-Personhood.git
cd ZK-based-Proof-of-Personhood
```

If already cloned:

```bash
git submodule update --init --recursive
```

### 1. Install Prerequisites

- **Rust** — https://rustup.rs
- **Circom** — https://docs.circom.io/getting-started/installation/

### 2. Compile Circuits

```bash
cd zkid/wallet-unit-poc/circom
```

See [`zkid/wallet-unit-poc/circom/README.md`](https://github.com/privacy-ethereum/zkID/blob/RSA-X.509-Cert/wallet-unit-poc/circom/README.md) for full compilation instructions.

### 3. Run E2E Tests

```bash
cd zkid/wallet-unit-poc/ecdsa-spartan2
```

See [`zkid/wallet-unit-poc/ecdsa-spartan2/README.md`](https://github.com/privacy-ethereum/zkID/blob/RSA-X.509-Cert/wallet-unit-poc/ecdsa-spartan2/README.md) for test instructions.

### 4. Start the Verifier Server

```bash
cd go-zkid-verifier
```

See [`go-zkid-verifier/README.md`](https://github.com/privacy-ethereum/go-zkid-verifier/blob/main/README.md) for configuration and launch instructions.

### 5. Run the Example Apps

**iOS:**

```bash
cd openac-taiwan-citizen-digital-certificate-ios-example
```

See [`openac-taiwan-citizen-digital-certificate-ios-example/README.md`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-ios-example/blob/main/README.md) for Xcode setup and simulator/device instructions.

**Android:**

```bash
cd openac-taiwan-citizen-digital-certificate-android-example
```

See [`openac-taiwan-citizen-digital-certificate-android-example/README.md`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-android-example/blob/main/README.md) for Android Studio setup and emulator/device instructions.

**Web:**

```bash
cd openac-taiwan-citizen-digital-certificate-web-example
pnpm install
pnpm fetch:assets   # downloads WASM + circuit bundles; copies to public/assets/
pnpm dev
```

See [`openac-taiwan-citizen-digital-certificate-web-example/README.md`](https://github.com/privacy-ethereum/openac-taiwan-citizen-digital-certificate-web-example/blob/main/README.md) for environment variables and deployment instructions.
