# AI audit limitations

## Purpose

This document records the limits of the initial UniStore porting and rebranding audit. It is intentionally explicit about what was inspected, what remains unverified, and which conclusions must not be treated as implementation facts.

## Audit context

The target is the Floflis fork of Ubuntu App Center. The repository is a Flutter/Dart workspace, not a single-file application. The available inspection confirmed the root workspace manifest and Snap packaging configuration, and reviewed the repository README and a recent upstream commit touching desktop launcher-title synchronization.

Confirmed observations:

- The root `pubspec.yaml` declares the workspace package name `app_center_workspace`, Dart SDK range `>=3.1.0 <4.0.0`, and Melos.
- `snap/snapcraft.yaml` packages the application as `snap-store`, builds the Linux Flutter target, and includes GTK development packages in its build dependencies.
- The Snap definition includes desktop integration, PackageKit integration, GNOME-related integration, and Ubuntu-specific plugs.
- The root README describes the product as App Center and documents public deep links including `snap://` and the `appcenter://` scheme.
- A recent upstream commit shows that the package's localization ARB files are used to synchronize desktop launcher titles, including an App Center Updates launcher.

These are confirmed examples, not an exhaustive inventory.

## Important limitation: this is not a complete source-tree audit

The inspection channel available for this pass did not provide a reliable recursive checkout or complete recursive Git tree. GitHub code search also did not return usable results for the attempted repository-scoped queries. Therefore this pass did **not** establish a complete list of files, packages, imports, generated files, assets, tests, CI workflows, translations, or native integrations.

In particular, the following have not been exhaustively inspected:

- every package under `packages/`;
- all Dart imports and conditional imports;
- every Flutter platform implementation;
- all references to Yaru, GTK, libadwaita, GNOME, Ubuntu, Snap, PackageKit, and desktop-only APIs;
- generated Dart, localization, code-generation, and build-runner outputs;
- all asset manifests, desktop files, D-Bus service files, AppStream metadata, and Snap interfaces;
- CI workflows, integration tests, golden tests, and release automation;
- transitive dependencies and their platform support;
- runtime behavior on a physical Android device.

Do not interpret an absence of a search result as proof that a symbol or dependency does not exist.

## Model and reasoning limitations

This audit was produced by a language model. Even when the model can reason carefully, a response is not equivalent to a complete, reproducible, extended-thinking engineering investigation or a locally executed build.

Possible blind spots include:

1. **Incomplete retrieval**: tools may expose selected files or snippets rather than the entire repository. A missing result may mean the file was not retrieved or indexed.
2. **Context-window loss**: details from large files, long histories, or many packages may be compressed, omitted, or confused.
3. **False negatives in naming searches**: integrations can be hidden behind aliases, generated code, dependency APIs, reflection, build scripts, native code, or indirect imports.
4. **False positives in text searches**: a word such as GTK may occur in documentation or a test fixture without being a runtime dependency.
5. **Generated-code ambiguity**: generated files may be checked in, regenerated during CI, or produced differently on each platform.
6. **Dependency-transitivity gaps**: a package may not import GTK directly while a dependency or platform plugin does.
7. **Platform lifecycle gaps**: Android may compile while failing at runtime because of permissions, intents, background restrictions, storage rules, or missing native services.
8. **Build-environment mismatch**: a successful analysis based on manifests does not prove compatibility with the project's exact Flutter, Dart, Gradle, Android SDK, NDK, Java, or Kotlin versions.
9. **Upstream drift**: this repository may diverge from Ubuntu upstream. Upstream assumptions cannot safely be applied without comparing the fork's current tree and history.
10. **Branding and protocol compatibility**: replacing visible product text is not the same as safely renaming package IDs, URI schemes, desktop IDs, Snap names, D-Bus names, or persisted settings.

## Required verification before claiming a complete UniStore rename

Run a full clone-based inventory from the target branch and retain the command output in the PR or CI artifacts. At minimum:

```bash
git grep -n -i -E 'app[ _-]?center|appcenter|yaru|gtk|gnome|ubuntu|packagekit' -- .
find . -type f | sort
find packages -name pubspec.yaml -print
```

Also inspect generated and ignored files where relevant, and search case variants and identifiers separately. Classify every match as:

- user-visible branding;
- package/module identifier;
- public URI/deep-link protocol;
- platform-specific integration;
- upstream provenance/license/attribution;
- test fixture or historical documentation;
- generated artifact.

Then run the repository's documented formatting, static analysis, unit, integration, and platform build commands. Record exact tool versions and results.

## Naming policy proposed for UniStore

Use **UniStore** as the user-facing product name. Do not blindly replace every occurrence of App Center.

| Existing identifier | Proposed treatment |
|---|---|
| Visible product title and user-facing copy | Rename to UniStore after localization review |
| Dart package and import identifiers such as `app_center` | Rename only as a coordinated package migration, updating all imports, workspace references, scripts, and generated files |
| Internal symbols such as `appCenterLabel` | Rename only if part of the public/internal API migration scope |
| `appcenter://` deep-link scheme | Preserve initially for backward compatibility; introduce a new scheme only with dual-scheme parsing and migration tests |
| Snap package name `snap-store` | Keep separate from product branding; it is a distribution identifier, not automatically a rename target |
| D-Bus names, desktop file IDs, service names | Treat as integration contracts; change only with compatibility aliases and migration tests |
| Upstream copyright, license, attribution, and provenance | Never rename or erase in a way that misrepresents origin |

## Current completion status

This document is an audit-boundary record, not a certification of completion. The present change adds documentation and an Android migration plan. It does not claim that every App Center reference has been renamed, that Yaru/GTK has been removed, or that Android already builds.

A full source-level rebrand should be a separate implementation change after the complete inventory is available.
