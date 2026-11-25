# UniStore provider architecture

## Status and scope

This document proposes the target architecture for an extensible, cross-platform software store. It is a design contract for future implementation, not a claim that these providers or interfaces already exist in this repository.

The goal is to recover the useful modularity of the GNOME Software plugin model without depending on GNOME Software, GTK, Yaru, or its internal APIs. UniStore owns its contracts and user experience; ecosystem integrations implement those contracts behind adapters.

Initial target ecosystems:

- Linux: Flatpak, Snap, and APT/DEB where the host exposes a supported installation backend.
- Android: APK-based applications, installed through Android's user-mediated package installation facilities.
- Future ecosystems: add providers without coupling the shared catalog or UI to their package-manager details.

A platform does not have to support every ecosystem. Unsupported operations must be reported explicitly, not simulated.

## Terminology

These terms describe different responsibilities and must not be used interchangeably.

| Term | Meaning | Examples |
|---|---|---|
| Ecosystem / package format | The package and runtime model understood by an integration | Flatpak, Snap, DEB, APK |
| Source | A configured origin of catalog data or artifacts | Flathub, a Snap store channel, an APT repository, an F-Droid-compatible repository |
| Backend | A host service, API, or command interface that performs ecosystem operations | Flatpak installation, snapd, PackageKit, APT tooling |
| Installer | The platform mechanism that commits an installation | Flatpak transaction, snapd operation, Android PackageInstaller |
| Provider | UniStore's adapter for one ecosystem and its supported operations | FlatpakProvider, SnapProvider, AndroidApkProvider |
| Capability | An operation or feature a provider can actually perform in the current environment | Search, install, update, remove, permissions, rollback |

A repository/source adapter may be separate from the installation backend. For example, an Android catalog can read a repository index while Android's PackageInstaller handles the final APK installation. Likewise, PackageKit is a backend abstraction and does not, by itself, prove that Flatpak support is installed or configured.

## Architectural boundaries

Use a dependency direction that points inward:

1. **Presentation**: Flutter screens, navigation, accessibility, and user interaction.
2. **Application orchestration**: search, source management, install/update workflows, policy decisions, and progress presentation.
3. **Domain contracts**: provider interfaces, normalized app/package models, capability descriptions, operation states, and typed errors.
4. **Provider adapters**: Flatpak, Snap, APT/DEB, Android APK, and future integrations.
5. **Host integrations**: native APIs, daemons, command-line tools, repository clients, and platform installation facilities.

The shared domain and UI must not import Flatpak, Snap, PackageKit, APT, Android framework, GTK, or Yaru classes directly. Provider adapters may depend on their ecosystem libraries and platform-specific code.

## Provider contract

The provider API should be asynchronous, typed, capability-aware, and independent of Flutter. The following is illustrative Dart, not a finalized public API:

```dart
abstract interface class PackageProvider {
  /// Stable machine identifier, for example "flatpak" or "android-apk".
  String get id;

  /// Localized display name supplied by the presentation layer when possible.
  String get displayName;

  /// Ecosystem and host information used for matching and diagnostics.
  ProviderDescriptor get descriptor;

  /// Features actually supported in the current environment.
  Future<Set<ProviderCapability>> getCapabilities();

  /// Sources currently configured for this provider.
  Future<List<PackageSource>> listSources();

  /// Search only sources and package scopes enabled by policy.
  Future<ProviderPage<PackageSummary>> search(
    PackageQuery query,
    PageRequest page,
  );

  /// Resolve a provider-native identifier into normalized details.
  Future<PackageDetails?> getDetails(PackageRef package);

  /// Return installed packages visible to this provider.
  Future<ProviderPage<InstalledPackage>> getInstalled(
    PageRequest page,
  );

  /// Start an operation only when the provider advertises the capability.
  Stream<PackageOperationEvent> execute(
    PackageOperation operation,
  );
}
```

The contract is a starting point. Do not force every ecosystem to implement meaningless methods. Capability checks and typed unsupported-operation results are mandatory; optional sub-interfaces may be preferable if the final API becomes too broad.

### Suggested capabilities

Capabilities should be granular rather than a single boolean called `supportsInstall`.

- `catalogSearch`, `packageDetails`, `screenshots`, `categories`
- `listSources`, `addSource`, `removeSource`, `refreshSource`
- `listInstalled`, `install`, `update`, `remove`
- `checkUpdates`, `downloadOnly`, `cancelOperation`
- `permissions`, `runtimeManagement`, `rollback`, `pinVersion`
- `verifyArtifact`, `verifyRepositoryMetadata`

A provider may expose different capabilities depending on the host, backend version, permissions, configured sources, or platform policy. The UI must query the runtime capability set and explain unavailable actions.

## Normalized models and identity

The shared catalog needs normalized models without erasing ecosystem-specific facts.

- `PackageRef`: provider ID plus provider-native package identifier and, when needed, source identity.
- `PackageSummary`: display name, summary, icon/artwork references, categories, and source attribution.
- `PackageDetails`: full description, screenshots, version choices, licenses, links, permissions, runtime requirements, and provenance.
- `PackageSource`: stable source ID, provider ID, display name, origin, enabled state, trust state, and refresh status.
- `InstalledPackage`: provider-native identity, installed version, installation scope, origin, and update state.
- `ProviderDescriptor`: provider ID, API compatibility version, supported platforms, and diagnostic metadata.

Do not assume that two packages with the same display name are the same application. Do not assume a Flatpak app ID, Snap name, Debian package name, Android application ID, and catalog ID are interchangeable. Cross-provider matching must use explicit metadata and confidence, and must never silently merge identities.

Preserve provider-native metadata in a namespaced extension field where necessary. Normalization must not discard information needed for security review, troubleshooting, or advanced controls.

## Source management and trust

Sources are first-class objects. Users must be able to see where an app comes from and which provider will handle it.

- Show source name, origin, provider, enabled state, and trust/verification status.
- Explain whether trust comes from repository signatures, artifact signatures, a host package manager, or an explicit user decision.
- Verify repository metadata and downloaded artifacts according to the ecosystem's security model.
- Handle expired, revoked, malformed, replayed, or unavailable metadata explicitly.
- Do not silently add a source because a search result references it.
- Require clear confirmation before adding a third-party source or changing trust.
- Keep source identity separate from package identity.
- Never treat an arbitrary download URL as a repository or as trusted package metadata.

A provider must document what it verifies and what it delegates to the host. A successful download is not proof of authenticity.

## Operation lifecycle

Install, update, and removal are stateful operations, not fire-and-forget method calls.

Suggested states:

`queued → resolving → downloading → verifying → awaiting-user-action → installing → completed`

Terminal alternatives include `cancelled`, `failed`, and `requires-attention`. Not every provider uses every state. Events should include operation ID, package reference, timestamps, progress when measurable, and typed failure details.

- Support cancellation only when the backend can actually cancel safely.
- Distinguish download progress from installation progress.
- Do not invent percentages when the backend cannot report them.
- Surface authentication, privilege escalation, license prompts, system confirmations, and reboot requirements as explicit user actions.
- Make retries idempotent where possible and protect against duplicate operations.
- Refresh installed state after completion instead of assuming the backend succeeded.
- Record enough structured diagnostics to debug failures without logging secrets or sensitive paths unnecessarily.

## Initial provider matrix

This is a proposed scope matrix, not a statement that these integrations are implemented today.

| Provider | Target platform | Source/catalog | Installation backend | Notes |
|---|---|---|---|---|
| FlatpakProvider | Linux | Flatpak remotes, including Flathub when configured | Flatpak host tooling/API | Manage app refs, runtimes, remotes, scopes, and permissions where supported |
| SnapProvider | Linux | Snap store/channel metadata | snapd | Preserve channels, confinement, interfaces, and refresh semantics |
| AptProvider | Linux | Configured APT repositories and package metadata | APT or a supported host package service | Respect system package policy, dependencies, maintainer scripts, and privilege prompts |
| AndroidApkProvider | Android | A documented Android-compatible catalog/repository | Android PackageInstaller | User-mediated installation; package signatures and application IDs are central |
| F-Droid source adapter | Android | F-Droid-compatible repository metadata | AndroidApkProvider installer path | Treat repository parsing/signature verification as source functionality, not as a second installer |

### Flatpak

Flatpak should be a first-class Linux provider, not an incidental side effect of PackageKit support.

- Discover whether Flatpak tooling or a supported library/backend is present.
- Enumerate configured remotes and their scope (user/system).
- Search and resolve app refs using stable Flatpak identity.
- Model runtimes and extensions as dependencies, not ordinary unrelated apps.
- Respect per-user and system installation scopes.
- Expose permissions/overrides only when the selected backend can inspect and manage them safely.
- Handle remote priority, branch, architecture, runtime availability, and transaction errors.
- Do not assume Flathub is configured or that every Flatpak comes from Flathub.
- Do not claim Flatpak support merely because PackageKit is available.

The implementation decision between Flatpak CLI, libflatpak, or another supported integration belongs in a later technical spike. Compare maintainability, packaging, API stability, progress reporting, privilege handling, and portability before choosing.

### Snap

Snap remains a separate provider.

- Integrate with snapd rather than treating Snap as a Flatpak-like archive.
- Preserve channel, revision, confinement, interface, and refresh concepts.
- Report store and daemon availability independently.
- Keep Ubuntu-specific plugs and desktop integration in the Linux adapter.
- Do not expose Snap operations on Android.

### APT / DEB

APT/DEB support should be evaluated as a distinct provider.

- A DEB file format is not the same thing as an APT repository.
- Prefer a supported system package service or a carefully scoped native integration over invoking privileged commands from the UI.
- Respect dependency resolution, package triggers, maintainer scripts, held packages, pinning, and system upgrades.
- Distinguish installing a local DEB from searching configured APT repositories.
- Do not promise safe removal or rollback where the backend cannot guarantee it.

### Android APK

Android is not a Linux package manager port.

- The provider can retrieve catalog metadata and APK artifacts from documented sources.
- Android PackageInstaller is the system installation mechanism, with user confirmation and OS restrictions where required.
- Validate package name, version code, SDK constraints, ABI/splits, checksums, and signing identity.
- Updates require compatible signing identity and must respect Android package ownership.
- Installed-app visibility is subject to Android PackageManager visibility rules.
- Never imply that ordinary apps can silently install arbitrary APKs.
- Keep F-Droid-compatible repository parsing separate from the PackageInstaller adapter.

## Provider discovery and extension security

Start with built-in, statically registered providers. A dynamic third-party plugin system is a separate security and compatibility project.

- Providers declare stable IDs, supported platforms, API version, and capabilities.
- The registry rejects duplicate IDs and incompatible API versions.
- Provider initialization failures are isolated and visible in diagnostics; one broken provider must not crash the entire store.
- Providers are enabled only when platform, backend, and policy checks pass.
- Do not load arbitrary executable plugins from catalog sources.
- If third-party extensions are introduced later, define signing, provenance, permissions, sandboxing, update, revocation, and compatibility policies before enabling them.

The word "plugin" describes the modular architecture; it does not mean UniStore should execute untrusted downloaded code inside its process.

## Registry and orchestration

The application layer should depend on a provider registry, not on concrete provider classes:

```dart
abstract interface class ProviderRegistry {
  Future<List<ProviderDescriptor>> discover();
  Future<PackageProvider?> getProvider(String providerId);
  Future<Set<ProviderCapability>> capabilities(String providerId);
}
```

A coordinator can route a package operation by the provider ID inside `PackageRef`, validate capabilities, apply user/system policy, and stream normalized operation events. It must not guess a provider from an app's display name or URL.

Search aggregation should preserve provenance and provider identity. De-duplication is a presentation decision backed by explicit identity evidence, not a destructive merge in the provider layer.

## Platform policy

Provider availability is a combination of:

- compiled platform support;
- host operating system;
- installed backend/service and its version;
- configured sources;
- current permissions and authorization;
- security policy and user choices.

Examples:

- Linux with Flatpak installed may expose Flatpak operations even if Snap is absent.
- Linux with snapd available may expose Snap independently of Flatpak.
- Android exposes APK catalog/install capabilities, not Snap, Flatpak, or APT.
- A Linux build may share domain and catalog code with Android while excluding Linux native dependencies from the Android dependency graph.

## Suggested workspace boundaries

These are conceptual boundaries; map them against the real workspace before creating packages.

- `unistore_domain`: normalized models, contracts, capabilities, and operation states.
- `unistore_application`: provider registry, orchestration, policies, and use cases.
- `unistore_catalog`: source metadata, catalog indexing, search, and caching.
- `unistore_provider_flatpak`: Flatpak integration.
- `unistore_provider_snap`: Snap integration.
- `unistore_provider_apt`: APT/DEB integration.
- `unistore_provider_android_apk`: Android APK catalog and installation integration.
- `unistore_ui`: shared Flutter presentation.
- `unistore_platform_linux`: Linux host discovery and desktop integration.
- `unistore_platform_android`: Android platform integration.

Do not create all these packages immediately. First inventory the existing workspace and dependencies, then extract boundaries incrementally where they reduce coupling.

## Implementation sequence

1. Inventory the existing packages, dependency graph, and current Snap/PackageKit integration.
2. Define domain models, provider identity, capabilities, typed errors, and operation events.
3. Build a provider registry with test doubles and no native backend assumptions.
4. Wrap the existing Linux Snap integration behind a provider without changing behavior.
5. Add Flatpak as an independent Linux provider; validate user/system remotes, runtimes, permissions, and failures.
6. Evaluate and implement APT/DEB only after defining safe host privilege and package policy behavior.
7. Implement Android APK source/catalog and PackageInstaller path independently of Linux providers.
8. Add provider contract tests, backend integration tests, platform CI, and compatibility documentation.
9. Consider third-party dynamic extensions only after the built-in provider API is stable and its security model is reviewed.

## Testing requirements

- Contract tests shared by all providers, with capability-aware expectations.
- Provider-specific integration tests against disposable test sources or isolated environments.
- Tests for missing backend, daemon unavailable, no sources, stale metadata, invalid signatures, permission denial, cancellation, partial failure, and backend version mismatch.
- Identity tests proving that equal display names from different providers do not collapse into one package.
- Operation-state tests for monotonic transitions and terminal states.
- Android tests for user confirmation, signing mismatch, split APKs, package visibility, and cancellation.
- Linux tests for Flatpak remotes/scopes/runtimes, Snap channels/confinement, and APT dependency/privilege behavior.
- UI tests proving unavailable capabilities are hidden or explained accurately.

## Non-goals and guardrails

- Do not reimplement Flatpak, snapd, APT, or Android's package installer.
- Do not assume one universal backend can install every package format.
- Do not make PackageKit a prerequisite for every Linux provider.
- Do not treat a file extension as proof of package trust.
- Do not silently elevate privileges or bypass operating-system confirmation.
- Do not make Android depend on GTK, Yaru, GNOME Shell, Snap, or PackageKit.
- Do not claim a provider is supported until its backend detection, source handling, operations, error paths, and tests exist.

## Decision record

The architectural decision is to use **independent, capability-aware providers behind UniStore-owned interfaces**, with source/catalog adapters and host installers separated where their responsibilities differ.

This preserves a unified store experience while allowing each ecosystem to retain its own identity, security model, lifecycle, and platform constraints.
