# UniStore CLI backend and Iroh transport

## Status

This document records intended architecture. It does not claim that the `unistore` executable, Flutter bridge, Iroh integration, payment system, or publishing pipeline has been implemented.

## Product boundary

The `unistore` CLI is the canonical backend. Flutter is a client presentation layer. The backend owns domain logic, provider orchestration, source configuration, catalog access, compatibility, artifact verification, operations, persistent state, and structured diagnostics.

```text
Flutter UI
  └─ platform bridge / versioned client protocol
       └─ UniStore backend (command: unistore)
            ├─ universal catalog and domain services
            ├─ source/provider registry
            ├─ compatibility and policy engine
            ├─ artifact verification and staging
            ├─ transport abstraction (Iroh, HTTPS, native ecosystem)
            └─ target deployment adapters and host integrations
```

Flutter must not independently reimplement backend business rules. The CLI remains useful without Flutter for scripting, automation, diagnostics, and headless use.

## CLI and machine protocol

Provide human-friendly commands and a separate stable machine interface. Flutter must not parse terminal prose or build shell command strings from user/catalog input. Define protocol/API version negotiation, structured request/response envelopes, typed errors, stable IDs, event streams, pagination, capability reporting, safe cancellation where supported, and redaction rules for credentials, tokens, paths, and transport metadata.

A local process/stdio protocol is a reasonable desktop starting point, but keep it transport-independent. Android must not be assumed to support arbitrary external executable spawning: evaluate embedding the same backend logic via native library/FFI or a managed in-app service. Platform bridges may differ while preserving protocol semantics and backend ownership. The backend owns configuration, cache, database, staging, downloads, and logs; Flutter accesses them through backend operations.

## Iroh integration

Iroh is an optional peer-to-peer artifact transport, not a package manager, catalog, payment processor, entitlement authority, or publisher trust root. Use it where useful for large game builds, downloadable content, asset libraries, mods, maps, resource packs, and creator-distributed artifacts. Keep HTTPS and ecosystem-native transport options.

Backend-controlled lifecycle: resolve exact catalog item/version; validate source metadata and obtain an authenticated manifest; check entitlement/access policy where applicable; select Iroh or approved fallback; transfer into bounded staging; verify hashes and signatures against trusted manifest data; validate archive/content; then hand the verified artifact to its domain-specific deployment adapter.

Iroh addresses, tickets, or successful transfers are not proof of authenticity or entitlement. Keep stable content hashes and publisher identity separate from ephemeral transfer coordinates. Never deploy directly from an unverified stream.

## Game storefront and commerce

ImpossibleEngine games should be first-class UniStore products with cover/capsule art and cartridge-inspired presentation, screenshots, platform builds, version history, and optional demos, DLC, and bonus assets. Separate listing, offer, order/transaction, entitlement, artifact, and installation records. Payment confirmation comes from the commerce provider; artifact integrity is verified independently. Do not store card data or imply perpetual ownership, offline rights, DRM policy, or refunds unless the offer explicitly provides them.

A future ImpossibleEngine publishing workflow can produce signed release manifests and platform artifacts using stable product/game IDs. The engine must not depend on Flutter or the store UI.

## Security and failure behavior

- Backend is the policy enforcement point; UI cannot bypass it with arbitrary paths or shell fragments.
- Validate artifacts before extraction/deployment, including path traversal, symlink escape, decompression bombs, size, and target checks.
- Use least privilege and explicit user/system scope. Require OS confirmation/elevation where applicable.
- Treat drivers as high-risk privileged artifacts with strict device/kernel/OS matching, signature validation, recovery guidance, and conservative source policy.
- Keep provider, transport, and deployment failures isolated and typed.
- Preserve operation state and clean incomplete staging data safely after crashes/cancellation.
- Never log secrets, private transfer tickets, payment credentials, or unnecessary personal paths.

## Implementation sequence

1. Inspect the actual repository workspace and choose backend language/runtime based on existing code and platform constraints.
2. Define backend contracts, operation state, persistent-state ownership, and versioned machine protocol.
3. Implement a minimal `unistore` CLI for diagnostics, capabilities, and catalog queries with deterministic tests.
4. Connect Flutter as a thin desktop client; test backend absence, version mismatch, and structured errors.
5. Prototype Android embedding/bridge feasibility before claiming Android parity.
6. Add transport abstraction with local/mock and verified HTTPS paths.
7. Add Iroh behind that abstraction; test interruption, resumption where supported, hash mismatch, unavailable peers, and fallback.
8. Add one low-risk deployment domain and one game artifact workflow end-to-end.
9. Add commerce/entitlement integration only after seller, transaction, refund, privacy, and licensing boundaries are defined.

## Acceptance criteria

- Flutter contains no duplicate catalog/provider/install business logic.
- CLI machine protocol is versioned, documented, and contract-tested.
- Backend reports capabilities and unsupported operations explicitly.
- Iroh is optional and replaceable; artifacts are verified before deployment.
- Transport success, entitlement, artifact authenticity, and installation success are distinct states.
- Android backend packaging is validated on-device, not assumed from desktop behavior.
- Implementation claims are backed by tests on the relevant target.
