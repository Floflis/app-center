# UniStore Android roadmap

## Goal

Make UniStore a first-class Android application without depending on GTK, Yaru, GNOME Shell, Linux desktop services, or Ubuntu-only runtime facilities.

The current repository is a Flutter/Dart workspace and its Snap packaging configuration explicitly builds the Linux target and includes GTK development packages. That proves the Linux distribution path has native desktop requirements; it does not prove that the Dart application itself is inseparable from GTK. The Android work must isolate shared application logic from platform-specific presentation and deployment adapters.

## Product architecture

UniStore is a universal discovery, distribution, and management hub for typed digital items, not only an application/package manager. Its top-level catalog can include applications, browser extensions, Construct 2/3 add-ons, desktop themes, fonts, assets, developer tools, game content, and future domains.

See [Universal catalog architecture](UNISTORE_UNIVERSAL_CATALOG.md) for the item model, domain adapters, target compatibility, trust model, and implementation strategy.

## Guiding architecture

Separate the product into five layers:

1. **Catalog and domain layer**: typed items, identities, artifacts, sources, publishers, licenses, provenance, and compatibility metadata.
2. **Application orchestration**: search, collections, source management, install/update/configure workflows, policy decisions, and operation state.
3. **Domain integrations**: package providers, browser extension sources, Construct add-on catalogs, theme catalogs, and other domain-specific integrations.
4. **Target deployment adapters**: Android PackageInstaller, Linux package managers, browser-supported handoff/APIs, editor-specific deployment, and user-scoped theme/configuration deployment.
5. **Presentation**: shared Flutter experience with item-specific detail pages and platform-appropriate navigation, dialogs, accessibility, and interaction patterns.

Do not make Android emulate Linux desktop APIs. Keep GTK/Yaru imports and Linux-only dependencies behind Linux-specific adapters and conditional imports.

## Package providers as one domain

Flatpak, Snap, APT/DEB, and Android APK remain important, but they are one domain inside the larger catalog. See [Package provider architecture](PACKAGE_PROVIDER_ARCHITECTURE.md) for package-specific contracts, source trust, operations, and backend behavior.

- Linux providers: Flatpak, Snap, and potentially APT/DEB.
- Android provider: APK-based applications, installed through Android's PackageInstaller flow.
- Catalog/source adapters remain separate from host installation mechanisms where their responsibilities differ.
- Each provider advertises runtime capabilities; the UI must not expose unsupported actions.
- Package identity remains provider-native and is not inferred from display names.

## Phase 0: repository and dependency audit

- Clone the complete repository and enumerate every workspace package and package manifest.
- Map the dependency graph, direct and transitive dependencies, Flutter plugins, generated code, and native source.
- Search all Dart, YAML, Gradle, Kotlin, Java, C/C++, shell, desktop, Snap, and CI files for GTK, Yaru, GNOME, Ubuntu, PackageKit, Snap, Flatpak, APT, D-Bus, browser, extension, and theme integrations.
- Classify each dependency as shared, Linux-only, Android-compatible, replaceable, or unsupported.
- Inspect the upstream project and the fork's divergence before porting or deleting code.
- Record exact Flutter, Dart, Java, Gradle, Android Gradle Plugin, Kotlin, Android SDK, and NDK requirements.

**Exit criteria:** a reviewed package/dependency matrix and a platform integration map.

## Phase 1: universal catalog contracts

- Define CatalogItem, ItemKind, Artifact, Target, Source, Publisher, License, CompatibilityResult, and Provenance.
- Keep package-specific models under the application/package domain.
- Define domain-specific metadata extensions without forcing every item into package fields.
- Define provider and deployment-adapter registries separately.
- Add typed capabilities, operation states, cancellation semantics, and typed errors.
- Preserve provider-native identity and source attribution.
- Add schema versioning and migration rules for cached catalog data.

**Exit criteria:** catalog and deployment contracts pass unit tests with fake providers and adapters; no native backend is required.

## Phase 2: Android application skeleton and UI

- Add/enable the Flutter Android platform directory and Gradle configuration if absent.
- Define Android application ID, namespace, versioning, minimum SDK, target SDK, signing configuration, and release flavors.
- Keep signing secrets outside the repository and inject them through CI secrets or local environment configuration.
- Configure app icon, adaptive icon, splash screen, display name, theme, status/navigation bars, and edge-to-edge behavior.
- Build responsive discovery, search, collections, and item-detail experiences.
- Show only item domains and actions supported on Android; do not surface Linux-only theme or package deployment actions as installable Android features.
- Use mobile navigation, back handling, safe areas, keyboard-aware forms, and accessible touch targets.

**Exit criteria:** an installable Android APK/AAB launches without GTK/Yaru or Linux native libraries and can render mock catalog items from multiple domains.

## Phase 3: domain and source integrations

- Implement catalog/source adapters independently from target deployment.
- Start with documented Android-compatible application sources, such as curated F-Droid-compatible repositories or a first-party catalog.
- Prototype browser-extension discovery with official store handoff; direct installation is only allowed through supported browser APIs and policies.
- Add Construct 2 and Construct 3 as distinct catalog targets with editor-specific metadata and compatibility.
- Add theme catalogs for GTK, Metacity, icon/cursor themes, and Omarchy as Linux targets.
- Record publisher, license, source provenance, target version, artifact checksums/signatures, and requested permissions.
- Never treat arbitrary URLs or archives as trusted repositories.

**Exit criteria:** the catalog can display typed items from multiple domains with clear compatibility, provenance, and installation-method labels.

## Phase 4: target deployment adapters

- Android applications: use Android PackageInstaller with required user confirmation and signature checks.
- Linux applications: use independent Flatpak, Snap, and potentially APT/DEB package providers.
- Browser extensions: use official browser-supported APIs or verified store handoff; never bypass browser security or policy.
- Construct add-ons: deploy only through verified editor-supported paths; keep Construct 2 and Construct 3 compatibility separate.
- GTK/Metacity/Omarchy themes: validate archives, deploy user-scoped by default, back up current configuration, and provide revert where possible.
- Other domains: add a deployment adapter only after defining target compatibility, validation, permissions, and recovery.

**Exit criteria:** each advertised integrated deployment path passes target-specific tests. Discovery-only/manual paths are labeled honestly.

## Phase 5: Android installation and lifecycle

Android intentionally restricts silent installation by ordinary applications.

- Present system confirmation UI when required; do not promise silent installs.
- Handle unknown-source install permission as a user-controlled setting.
- Validate APK signatures, package names, version codes, SDK compatibility, ABI, and split APK requirements.
- Handle install sessions, cancellation, progress, failure reasons, and post-install verification.
- Respect Android package visibility, scoped storage, WorkManager, Doze, notification permissions, and lifecycle constraints.
- Never request device-owner privileges or accessibility-service access as a workaround for ordinary store limitations.

**Exit criteria:** tested user-mediated installation and Android lifecycle behavior on supported API levels.

## Phase 6: Linux provider implementation

- Wrap the existing Snap integration behind a provider without changing established behavior.
- Add Flatpak as an independent provider, with explicit detection of backend availability and configured remotes.
- Evaluate APT/DEB separately, including dependency resolution, privilege handling, and system package policy.
- Add GTK/Metacity/Omarchy theme deployment as a separate domain adapter, not as a package-manager provider.
- Keep PackageKit optional and provider-specific; its presence must not imply Flatpak support.
- Ensure one missing backend does not disable unrelated domains or providers.

**Exit criteria:** each implemented provider/adapter passes its contract and integration tests independently.

## Phase 7: security, tests, and release quality

- Validate source metadata and artifact signatures/checksums where available.
- Reject path traversal, symlink escapes, decompression bombs, and writes outside the target.
- Do not execute install scripts by default; require explicit consent for executable add-ons and scripts.
- Back up user profiles/configuration before modifications and test restore/revert.
- Test malformed, incompatible, revoked, interrupted, and malicious artifacts.
- Add provider/adapter contract tests, unit tests, widget tests, integration tests, static analysis, APK/AAB builds, and Linux CI.
- Complete privacy, security, licensing, and repository-policy reviews before release.

**Exit criteria:** documented test matrices pass and each domain's support level is accurately represented.

## Suggested package boundaries

These are conceptual names, not claims that packages already exist:

- `unistore_domain`: universal catalog models, identity, compatibility, provenance.
- `unistore_application`: orchestration, policy, registries, and use cases.
- `unistore_catalog`: source adapters, indexing, search, and caching.
- `unistore_ui`: shared Flutter presentation.
- `unistore_domain_packages`: package-specific models and contracts.
- `unistore_provider_flatpak`, `unistore_provider_snap`, `unistore_provider_apt`.
- `unistore_provider_android_apk`.
- `unistore_domain_browser_extensions` and browser-specific adapters.
- `unistore_domain_construct_addons` with separate Construct 2 and Construct 3 targets.
- `unistore_domain_themes` with GTK, Metacity, Omarchy, and future theme adapters.
- `unistore_platform_linux`, `unistore_platform_android`.

Do not create every package at once. Extract boundaries from the real workspace incrementally.

## Key architectural rule

The universal catalog describes **what an item is**. Domain integrations describe **where it comes from and what it means**. Target deployment adapters describe **how it can be safely applied**. The UI orchestrates these contracts without embedding ecosystem-specific behavior.

This keeps Android independent of Linux desktop technology while allowing UniStore to grow far beyond traditional application packages.
