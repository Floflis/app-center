# UniStore Android roadmap

## Goal

Make UniStore a first-class Android application without depending on GTK, Yaru, GNOME Shell, Linux desktop services, or Ubuntu-only runtime facilities.

The current repository is a Flutter/Dart workspace and its Snap packaging configuration explicitly builds the Linux target and includes GTK development packages. That proves the Linux distribution path has native desktop requirements; it does not prove that the Dart application itself is inseparable from GTK. The Android work must isolate shared application logic from platform-specific presentation and package-management backends.

## Guiding architecture

Separate the product into four layers:

1. **Domain and catalog layer**: package metadata, app identity, versions, screenshots, descriptions, categories, ratings, and update state. No Flutter widgets or operating-system APIs.
2. **Application services**: search, filtering, sorting, recommendations, install/update orchestration, downloads, and state management behind interfaces.
3. **Platform adapters**: Android, Linux/Snap, and any future platforms implement the interfaces using native platform capabilities.
4. **Presentation**: shared Flutter screens and design tokens, with platform-appropriate navigation, dialogs, accessibility, and interaction patterns.

Do not make Android emulate Linux desktop APIs. Keep GTK/Yaru imports and Linux-only dependencies behind Linux-specific adapters and conditional imports.

## Phase 0: repository and dependency audit

- Clone the complete repository and enumerate every workspace package and package manifest.
- Map the dependency graph, direct and transitive dependencies, Flutter plugins, generated code, and native source.
- Search all Dart, YAML, Gradle, Kotlin, Java, C/C++, shell, desktop, Snap, and CI files for GTK, Yaru, GNOME, Ubuntu, PackageKit, Snap, D-Bus, and desktop-only APIs.
- Classify each dependency as shared, Linux-only, Android-compatible, replaceable, or unsupported.
- Inspect the upstream project and the fork's divergence before porting or deleting code.
- Record exact Flutter, Dart, Java, Gradle, Android Gradle Plugin, Kotlin, Android SDK, and NDK requirements.

**Exit criteria:** a reviewed package/dependency matrix and a platform integration map. No dependency is removed merely because its name looks desktop-specific.

## Phase 1: Android application skeleton

- Add/enable the Flutter Android platform directory and Gradle configuration if it is absent.
- Define Android application ID, namespace, versioning, minimum SDK, target SDK, signing configuration, and release flavors.
- Keep signing secrets outside the repository and inject them through CI secrets or local environment configuration.
- Configure app icon, adaptive icon, splash screen, display name, theme, status/navigation bars, and edge-to-edge behavior.
- Add Android-specific launcher metadata and required manifest declarations.
- Verify debug and release builds on a clean CI runner.

**Exit criteria:** an installable Android APK/AAB that launches into a placeholder UniStore screen without loading GTK/Yaru or Linux native libraries.

## Phase 2: remove desktop-only UI coupling

- Find Yaru widgets, theme extensions, icon packages, GTK bindings, and GNOME-specific components.
- Replace desktop-only widgets with Flutter Material 3 or a platform-neutral UniStore component library.
- Preserve a consistent UniStore visual identity through shared color, typography, spacing, shape, and icon tokens rather than copying the Ubuntu/Yaru theme.
- Use responsive layouts: compact phone, expanded phone/foldable, and tablet.
- Replace desktop window assumptions with mobile navigation, back handling, safe areas, keyboard-aware forms, and touch targets.
- Add screen-reader semantics, scalable text, contrast checks, and reduced-motion considerations.
- Keep Linux-specific widgets in Linux-only packages or conditional imports.

**Exit criteria:** Android dependency resolution and compilation do not include GTK/Yaru libraries, and core screens render correctly on phone and tablet sizes.

## Phase 3: package source and repository backend

This is the largest product decision. The Linux App Center's Snap/PackageKit integration cannot simply be called from Android.

- Define a backend-neutral repository API for catalog search, package details, screenshots, categories, versions, changelogs, and update metadata.
- Define package-source types explicitly. A Snap source is not an Android package source.
- Select Android-compatible sources, such as curated F-Droid-compatible repositories, a first-party catalog, or another legally and operationally suitable repository format.
- Implement repository index parsing, signature verification, trust roots, freshness checks, and safe error handling.
- Show source provenance, signing identity, permissions, version, architecture, and compatibility before installation.
- Support repository add/remove/enable/disable with clear trust prompts.
- Never treat an arbitrary APK URL as a trusted repository.

**Exit criteria:** UniStore can display a useful Android catalog from a documented, verifiable repository source without Snap Store or PackageKit.

## Phase 4: installation and update model

Android intentionally restricts silent installation by ordinary applications. Design around those platform rules.

- Use Android's package installation flow (for example, PackageInstaller) where appropriate and permitted.
- Present the system confirmation UI when required. Do not promise silent installs or unrestricted updates.
- Handle unknown-source install permission as a user-controlled Android setting, with an explanation and a return-to-app flow.
- Validate APK signatures, package names, version codes, SDK compatibility, ABI, and split APK requirements before install.
- Handle install sessions, cancellation, progress, failure reasons, and post-install verification.
- Determine whether updates can be applied by the same signing identity and package ownership.
- Explain that apps installed by Google Play or another store may not be updateable by UniStore if signing identities differ.
- Treat uninstall, downgrade, and permission changes as separate privileged/user-mediated operations.
- Do not request device-owner privileges or accessibility-service access as a workaround for ordinary store limitations.

**Exit criteria:** a tested, user-mediated install/update path with accurate reporting of Android restrictions and signature conflicts.

## Phase 5: Android-specific capabilities

- App details: deep links to Android app details/settings where useful.
- Installed-app inventory: use supported PackageManager visibility APIs and declare only justified package visibility queries.
- Downloads: use app-private storage, resumable downloads where appropriate, checksum verification, and clear cleanup policies.
- Background work: use WorkManager for deferrable work and respect Doze, battery optimization, and network constraints.
- Notifications: request notification permission where applicable and keep update notifications opt-in and meaningful.
- Network: handle offline mode, captive portals, metered networks, TLS failures, and repository timeouts.
- Storage: avoid broad storage permissions; use scoped storage and system pickers when user-selected files are needed.
- Security: validate metadata, sanitize rendered text/HTML, prevent path traversal, and never execute downloaded content.
- Privacy: disclose telemetry, minimize data collection, and provide a clear policy.

**Exit criteria:** Android lifecycle, permissions, networking, storage, and background behavior are tested on supported API levels.

## Phase 6: tests and release quality

- Unit-test repository parsing, signature/trust validation, version comparison, compatibility checks, and install-state transitions.
- Add widget/golden tests for compact and expanded layouts.
- Add integration tests for search, details, source management, download, install handoff, cancellation, and update checks.
- Test API-level boundaries, at least one older supported Android version, current Android, and multiple ABIs where relevant.
- Test clean install, upgrade, interrupted download, low storage, offline state, revoked/invalid signatures, and conflicting package signatures.
- Add CI jobs for formatting, static analysis, tests, APK build, and AAB build.
- Publish reproducible build metadata and release checksums.
- Complete privacy, security, licensing, and repository-policy reviews before public release.

**Exit criteria:** CI produces installable release artifacts and the documented test matrix passes.

## Phase 7: Linux and Android coexistence

- Keep Linux/Snap as a platform adapter rather than deleting the existing distribution path.
- Retain GTK/Yaru dependencies only in the Linux-specific build graph.
- Keep Snap, PackageKit, D-Bus, desktop-entry, and Ubuntu-driver integration out of Android build targets.
- Share catalog/domain logic only where semantics genuinely match.
- Use platform capability interfaces and explicit unsupported-operation states.
- Document platform feature parity; do not imply that Android can install Snap packages.

## Suggested package boundaries

Names below are architectural proposals, not claims that these packages already exist:

- `unistore_domain`: entities, value objects, repository contracts, and use cases.
- `unistore_catalog`: source abstractions, metadata models, search, and catalog caching.
- `unistore_ui`: shared Flutter components and design tokens.
- `unistore_android`: Android repository adapters and PackageInstaller integration.
- `unistore_linux`: Snap/PackageKit/D-Bus/desktop adapters and Yaru/GTK-specific code.

Avoid splitting into many packages until the current workspace inventory shows a clear boundary and dependency direction.

## Key architectural rule

The shared application layer must depend on UniStore interfaces, not on GTK, Yaru, Snap, PackageKit, D-Bus, or Android framework classes. Platform implementations depend inward on those interfaces. This keeps the Android app independent of Linux desktop technology without breaking the Linux edition.
