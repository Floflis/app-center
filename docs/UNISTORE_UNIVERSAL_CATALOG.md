# UniStore universal catalog architecture

## Vision

UniStore is not merely a package manager with several package formats. It is a **universal discovery, distribution, and management hub for digital extensions and customizations** across operating systems, applications, creative tools, and desktop environments.

The store should be able to discover, present, validate, install, update, configure, disable, and remove many kinds of digital items while respecting the rules of the target application or platform.

Examples include:

- Operating-system applications: Flatpak, Snap, APT/DEB, Android APK.
- Browser extensions: extensions for compatible browsers and their official extension ecosystems.
- Construct add-ons: Construct 2 and Construct 3 plugins, behaviors, effects, exporters, templates, and related add-on types supported by each editor.
- Desktop themes: GTK themes, Metacity window themes, icon themes, cursor themes, GNOME Shell themes/extensions, and Omarchy configurations/themes.
- Developer tools: IDE/editor extensions, language servers, themes, snippets, templates, and toolchains.
- Creative tools: plugins, brushes, presets, scripts, templates, fonts, and asset libraries.
- Games: mods, maps, resource packs, add-ons, servers, and user-created content where the target game supports safe installation.
- Personalization: wallpapers, sounds, fonts, icons, cursors, widgets, shell prompts, terminal themes, and configuration bundles.

This list is deliberately open-ended. New item families should be addable through explicit domain models and adapters, not by adding special cases to the main UI.

## Core product model

The top-level concept is a **CatalogItem**, not a package. A catalog item represents something a user can discover. It may have one or more deliverable artifacts and one or more installation targets.

Suggested conceptual model:

- **CatalogItem**: user-facing identity, title, description, publisher, categories, media, links, license, and provenance.
- **ItemKind**: semantic family such as application, browser extension, editor add-on, theme, font, game mod, or asset pack.
- **Artifact**: a concrete downloadable or installable payload, including format, version, checksums, signatures, and compatibility metadata.
- **Target**: the application, OS, desktop environment, browser, editor, game, or user profile that receives the artifact.
- **Source**: origin of catalog metadata or artifact, including trust and update policy.
- **Provider**: an integration that discovers catalog entries or communicates with an ecosystem.
- **Installer / DeploymentAdapter**: the target-specific mechanism that places, registers, enables, or removes an artifact.
- **Capability**: a feature supported by a provider/adapter in the current environment.
- **Operation**: a tracked install, update, enable, disable, configure, or removal workflow.

One CatalogItem can have multiple artifacts, targets, or providers. For example, a theme might offer GTK and Metacity components in one bundle, while an editor add-on might publish separate compatible builds for Construct 2 and Construct 3.

Do not confuse a catalog provider with an installer. A browser store API may provide metadata while a browser-specific deployment adapter handles installation. A theme catalog may provide archives while a theme adapter validates and deploys them to the correct user directory.

## Domains and adapters

Organize the architecture around **item domains** and **target adapters**, rather than one giant provider interface.

### 1. Application/package domain

Examples: Flatpak, Snap, APT/DEB, Android APK.

- Linux package providers: Flatpak, Snap, APT/DEB.
- Android application provider: APK catalog/source integration plus Android PackageInstaller.
- Preserve each ecosystem's identity, dependency model, permissions, trust, and update rules.

See [Package provider architecture](PACKAGE_PROVIDER_ARCHITECTURE.md) for the package-specific contracts and implementation plan.

### 2. Browser extension domain

Examples: compatible extensions for Firefox-family browsers and Chromium-family browsers.

Potential features:

- Search and display extension metadata, publisher, permissions, version, compatibility, ratings, and source.
- Link to official extension store pages where direct installation is not supported.
- Use official browser APIs or store-supported flows when available.
- Detect browser family, version, profile, and extension-management capabilities only through supported interfaces.
- Explain permission changes before installation or update.
- Distinguish browser extensions, themes, dictionaries, search providers, and native companion applications.
- Never silently install by copying files into browser profiles or modifying protected browser state.

Direct installation depends on browser policy, signing, store APIs, enterprise restrictions, and platform. If the browser does not expose a supported installation mechanism, UniStore should provide discovery and a verified handoff to the official store rather than bypassing policy.

### 3. Construct editor add-on domain

Support add-ons for Construct 2 and Construct 3 as distinct targets with explicit compatibility metadata.

Potential item kinds include:

- Plugins
- Behaviors
- Effects
- Project templates
- Example projects
- Exporters or build-related extensions where supported
- Other editor-recognized add-on types

Requirements:

- Model Construct 2 and Construct 3 separately. Do not assume their formats, APIs, packaging, or compatibility are interchangeable.
- Preserve author, license, version, editor compatibility, dependencies, and source provenance.
- Validate archive structure and expected manifests before deployment.
- Install into a user-selected or documented editor add-on location only when the target editor supports that workflow.
- Handle enable/disable and removal without deleting user project data.
- Treat executable scripts and plugin code as untrusted until provenance and user consent are clear.
- Where editor APIs or installation locations are undocumented or unsupported, provide a download/manual-install workflow with clear instructions rather than pretending to have integrated installation.

Construct 2 and Construct 3 are separate integration targets and should have separate adapters if their installation or metadata behavior differs.

### 4. Desktop theme and appearance domain

Support themes and configuration packages for specific desktop environments and window managers.

Examples:

- GTK themes, with GTK version compatibility recorded.
- Metacity window themes.
- Icon themes and cursor themes.
- GNOME Shell themes and extensions, where supported.
- Omarchy themes/configuration bundles, with explicit version and dependency metadata.
- Other desktop environments and compositors as future targets.

Requirements:

- Model each component separately even when distributed in a bundle. A theme may include GTK, Metacity, icons, cursors, fonts, and wallpapers.
- Record target desktop environment, toolkit/version, window manager/compositor, display server assumptions, and dependencies.
- Prefer user-scoped installation. System-wide changes require explicit authorization and a clear explanation.
- Preview assets before applying them.
- Validate archive paths and reject path traversal, symlink escapes, unsafe hooks, and unexpected executable payloads.
- Treat shell scripts and configuration bundles as code, not harmless images.
- Support backup/restore or a reversible deployment plan before changing active settings.
- Never assume a GTK theme is compatible with every GTK version or desktop environment.
- Omarchy is a configuration ecosystem, not simply a GTK theme. Its adapter must understand the target layout, dependencies, and versioning rather than copying files blindly.

### 5. General extension and asset domains

The same architecture can support:

- IDE/editor extensions, themes, snippets, and language tooling.
- Fonts, icon packs, cursor packs, wallpapers, sound packs, and desktop widgets.
- Terminal themes, shell prompts, dotfile bundles, and configuration profiles.
- Creative application plugins, brushes, presets, templates, and asset packs.
- Game mods, maps, resource packs, and add-ons where the target exposes a safe deployment mechanism.

Each new domain must define its own item kinds, target compatibility, artifact validation, installation semantics, update behavior, permissions, and rollback strategy. Do not treat every ZIP file as a generic installable extension.

## Domain contract and deployment adapter

The following Dart sketch is illustrative. It is not a finalized API and must be checked against the repository's actual workspace and language constraints.

```dart
abstract interface class CatalogProvider {
  String get id;
  ProviderDescriptor get descriptor;
  Future<Set<ProviderCapability>> getCapabilities();
  Future<ProviderPage<CatalogItem>> search(
    CatalogQuery query,
    PageRequest page,
  );
  Future<CatalogItemDetails?> getDetails(CatalogItemRef item);
}

abstract interface class DeploymentAdapter {
  String get id;
  TargetDescriptor get target;
  Future<Set<DeploymentCapability>> getCapabilities();
  Future<CompatibilityResult> checkCompatibility(Artifact artifact);
  Stream<DeploymentEvent> deploy(
    Artifact artifact,
    DeploymentAction action,
  );
}
```

A package provider can implement both catalog and deployment responsibilities or compose separate adapters. Other domains should not be forced to implement package-specific operations such as dependency resolution or system package removal.

Suggested deployment actions include:

- install / update / remove
- enable / disable
- apply / revert
- configure / reset
- import / export

Only expose actions that make sense for the item domain and target.

## Compatibility and target resolution

Compatibility must be evaluated before deployment, not inferred from a category label.

A target descriptor may include:

- OS and version
- CPU architecture
- desktop environment, toolkit, window manager, or compositor
- browser family and version
- editor/game/application name and version
- profile or installation scope
- required APIs, runtimes, dependencies, and feature flags

Compatibility results should be explicit:

- compatible
- compatible with warnings
- incompatible
- unknown / not enough information

Unknown compatibility must not be silently treated as compatible. The UI should explain the reason and offer a manual path only when it is safe to do so.

## Catalog, search, and user experience

UniStore should offer a unified discovery experience without pretending every item has the same lifecycle.

Suggested top-level navigation:

- Discover
- Applications
- Extensions
- Themes & Appearance
- Creative & Developer Tools
- Games & Content
- Installed / Managed
- Sources

Categories and filters should be data-driven. An item can appear in multiple collections without duplicating its identity.

Search results should show item kind, target, compatibility, publisher, source, license, and installation method. Details pages should adapt to the item domain: package permissions for applications, browser permissions for extensions, editor compatibility for Construct add-ons, and screenshots/previews plus target versions for themes.

Installed/managed items should include items deployed by UniStore and, where supported and authorized, items discovered from external managers. Clearly distinguish "managed by UniStore", "detected", and "external/manual" states.

## Security and trust model

The larger scope increases the attack surface. Treat all downloaded content as untrusted until validated.

- Verify source metadata and artifact checksums/signatures where the ecosystem provides them.
- Display publisher, source, license, version, compatibility, and requested permissions.
- Validate archive contents before extraction; prevent path traversal, symlink escapes, decompression bombs, and writes outside the selected target.
- Do not execute install scripts by default.
- If a domain requires scripts or executable plugins, show the code/provenance and require explicit user consent; prefer sandboxing where technically possible.
- Keep secrets and credentials out of logs.
- Make install/update/remove operations auditable and reversible where feasible.
- Preserve backups before changing user themes, profiles, editor directories, or configuration files.
- Separate user-level and system-level deployment, and never silently elevate privileges.
- Support revocation and removal of malicious or withdrawn catalog entries.
- Avoid loading third-party UniStore code in-process. Catalog providers and deployment adapters should be built-in or distributed through a separately secured extension mechanism.

## Plugin architecture for UniStore itself

There are two different meanings of "plugin":

1. **UniStore integration plugins/providers**: code maintained and shipped as part of UniStore (initial model).
2. **Items sold/distributed by UniStore**: browser extensions, Construct add-ons, themes, fonts, mods, and other content.

Do not conflate them. Installing a browser extension must not install a UniStore provider, and adding a catalog source must not execute downloaded code inside UniStore.

A future third-party provider SDK would require its own signing, sandbox, permission, API compatibility, review, revocation, and update model. It is not required to achieve a broad catalog.

## Implementation strategy

Build the universal model incrementally; do not attempt every ecosystem at once.

1. Audit the existing repository and map current app/package concepts before renaming or replacing them.
2. Define generic catalog entities, item kinds, artifact metadata, targets, compatibility results, sources, and provenance.
3. Keep the existing package-provider API as a specialized application/package domain, not the universal top-level abstraction.
4. Add deployment adapter contracts and operation lifecycle with test doubles.
5. Preserve the existing Snap path behind its package provider.
6. Prototype one non-package domain with a low-risk user-scoped deployment, such as a theme bundle with validation and backup/restore.
7. Prototype browser-extension discovery with official store handoff before attempting direct browser integration.
8. Add Construct 2 and Construct 3 metadata/discovery adapters, then deployment only where editor-supported paths are verified.
9. Add Flatpak independently, then evaluate APT/DEB and other package providers.
10. Add further domains only with compatibility, trust, installation, update, removal, and recovery tests.

Suggested first vertical slices:

- **Theme slice:** catalog entry → target compatibility → archive validation → user-scope deployment → backup/revert.
- **Browser slice:** catalog entry → browser compatibility → permission/source details → official store handoff.
- **Construct slice:** add-on metadata → C2/C3 compatibility → validated download → editor-specific install or documented manual handoff.

## Acceptance criteria for a new domain

A domain is not "supported" merely because UniStore can list or download its files. Before advertising integrated support, it must have:

- a documented item model and stable identity;
- source provenance and license metadata;
- target and version compatibility checks;
- artifact validation and security review;
- a supported deployment mechanism;
- install/update/disable/remove behavior appropriate to the domain;
- progress, cancellation, and meaningful error reporting where applicable;
- backup/rollback or a clearly documented non-reversible limitation;
- tests for malformed, incompatible, malicious, and interrupted artifacts.

If only discovery or download works, label it accurately as discovery-only or manual installation.

## Non-goals and guardrails

- Do not make UniStore a universal script runner.
- Do not assume every item can be installed automatically.
- Do not collapse all domains into one generic ZIP installer.
- Do not bypass browser, editor, operating-system, or application security policies.
- Do not silently modify system-wide configuration.
- Do not confuse catalog content with executable UniStore plugins.
- Do not claim an integration is complete until its target-specific workflow and recovery behavior are tested.

## Architectural decision

UniStore's top-level abstraction is a **universal catalog of typed digital items**, with domain-specific providers and target-specific deployment adapters. Package managers are one important domain among many, not the definition of the entire product.

This allows UniStore to grow from an application store into a cross-platform hub for applications, extensions, add-ons, themes, tools, and digital content without sacrificing compatibility, provenance, or security.
