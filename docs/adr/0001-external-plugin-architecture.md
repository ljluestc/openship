# ADR 0001: External installable plugins (Phase 0)

- **Status:** Proposed. Needs maintainer acceptance before any code.
- **Issue:** [#521, RFC: External installable plugin architecture](https://github.com/oblien/openship/issues/521)
- **Scope of this document:** decisions only. No implementation ships with this ADR.

## Context

Openship has three ways to extend what it can deploy. None of them lets an outside author ship a versioned extension that an operator can install, pin, upgrade and roll back without a core PR.

| Existing piece | Where | What it already gives us |
| --- | --- | --- |
| Curated app catalog | `packages/core/src/apps/catalog/*.json`, `schema.ts`, `scripts/gen-catalog.ts` | A strict validator (`parseAppTemplate`), a `schemaVersion` / `minEngine` gate, and a published JSON Schema (`apps/web/public/app.schema.json`) |
| Custom apps | `modules/apps/custom-app.service.ts`, `custom_app_template` table | Per-org uploads checked by the same validator. Trust comes from provenance: `verified: false` is forced by the server, and built-in ids cannot be shadowed |
| Signed system modules | `packages/adapters/src/system/modules/verify.ts` | An ed25519 keyring (supports key rotation), per-asset sha256, fail-closed verification, and a dev-only escape hatch that is ignored in production |
| Install phases | `packages/core/src/apps/install-phases.ts` | A shared install stepper used by the API and the dashboard |

A custom app is already a plugin with exactly one app, no version, and no package. The cheapest plugin system that delivers real value generalizes custom apps. It does not add a code host.

## Decision summary

**Accept with cuts.** Build a declarative plugin system now (the RFC's Kind A). Defer capability packs (Kind B). Code-executing host extensions (Kind C) need their own RFC and are explicitly out of scope until the declarative pipeline has shipped and held up.

| # | Decision |
| --- | --- |
| D1 | v1 plugins are **declarative only**. Nothing from a plugin package is ever executed, `require`d or `eval`ed by the Openship API, worker or dashboard. |
| D2 | The only v1 contribution point is **`apps`**: one or more app templates, validated by the existing `parseAppTemplate`. |
| D3 | Plugins are **installed per organization**, like custom apps. Each (org, plugin id) has one active version. |
| D4 | The **lockfile is database state**: one row per install, plus immutable stored package bytes for each installed version. It can be exported as `plugins.lock.json`. |
| D5 | **The server decides trust.** There are three tiers (`unverified`, `signed`, `verified`), and the package cannot claim any of them. |
| D6 | Compatibility is gated by `engines.openship` (semver range) when installing and again at boot. A failing plugin is deactivated. It never takes the host down. |
| D7 | v1 install sources are an **uploaded zip** and an **HTTPS URL with a pinned sha256**. A registry is a static, signed index file, and arrives in a later phase. |
| D8 | Plugins **wrap** apps. They do not replace the curated catalog. Custom apps keep working and can later be shown as single-app local plugins. |
| D9 | Entry points arrive in this order: API + contracts + SDK, then the dashboard, then the CLI. |

## Decisions in detail

### D1: Declarative only

The RFC's own threat-model table explains why: a template is data, and code is a second product. Every v1 plugin package is JSON plus static assets. The containers an app template starts do run third-party images on the operator's server, but custom apps and the curated catalog already do the same thing, under the same in-container boundary. This decision adds no new privilege. The trust UI must still say plainly that installing a plugin can run third-party images (see D5).

### D2: Contribution points

| Point | v1 | Why |
| --- | --- | --- |
| `apps` (service templates) | **Yes** | It is the existing, validated, provenance-gated path. It is the whole of v1. |
| `notifications.channel` | Candidate for v1.1 | Possible only if it can be expressed as a declarative webhook template (URL pattern, headers, body template) with no code. Needs its own issue. |
| `install.phase` hooks | No | A hook is code. That makes it Kind C. |
| `ui.dashboard.panel` | No | Needs a CSP/iframe/postMessage design. First-party only until a separate RFC. |
| `api.route`, `jobs.cron`, `events.listener` | No | All are code. They make the plugin host part of the API's attack surface. Kind C RFC. |

Core owns the list of points. A manifest that declares an unknown point is **rejected**, not ignored, so an author never ships something they believe is active when it isn't.

### Package format

```text
<plugin>.zip
├── plugin.json            # required: identity, version, engines, contributes
├── apps/<appId>.json      # one per contributed app; same schema as catalog entries
├── assets/…               # optional: icons / screenshots (png, svg, webp, jpg)
├── README.md              # optional
├── CHANGELOG.md           # optional
└── LICENSE                # recommended
```

`plugin.json` (illustrative; Phase 1 publishes the real schema next to `app.schema.json`):

```json
{
  "$schema": "https://openship.io/plugin.schema.json",
  "manifestVersion": 1,
  "id": "acme.analytics-stack",
  "name": "Acme analytics stack",
  "version": "1.2.0",
  "publisher": "acme",
  "license": "MIT",
  "engines": { "openship": ">=1.40.0 <2.0.0" },
  "contributes": { "apps": ["apps/acme-plausible.json"] }
}
```

Rules:

- `id` is `publisher.name`, lowercase `[a-z0-9-]`, so it is globally unique without a registry.
- Every contributed app id must start with `<plugin id>.`, or be namespaced under it in some other way that Phase 1 picks. This makes shadowing a built-in id, or another plugin's app, impossible by construction. The existing built-in-id check stays as a second line of defence.
- `version` must be strict semver. Pre-releases are allowed but never chosen by "upgrade to latest".
- `manifestVersion` works like the catalog's `schemaVersion`: if it is newer than the host understands, the result is `schema-too-new`, never a partial parse.
- Package fields such as `verified`, `available` or a trust tier are **removed and never trusted** (D5).

### Archive safety (required for Phase 2, tested first)

The archive is rejected unless all of these hold:

- no absolute paths, no `..` segments, no backslashes, no NUL bytes
- no symlinks, hard links, device files or other non-regular entries
- no duplicate names, including names that only differ in case or Unicode normalization
- only allowlisted extensions (`.json`, `.md`, images under `assets/`)
- bounded size: 10 MB compressed, 50 MB uncompressed, 500 entries, and a per-entry compression-ratio cap against zip bombs
- the path still resolves inside the plugin root **after** extraction

JSON is parsed as data, with size and depth limits. Openship computes the sha256 of the **exact archive bytes**, and every integrity and pinning check uses that hash.

### D3, D4: Install model and lockfile

- **Table `plugin_install`**: `organizationId`, `pluginId`, `version`, `sha256`, `source` (`upload` | `url`), `sourceUrl?`, `trustTier`, `status` (`active` | `disabled` | `incompatible`), `installedBy`, `installedAt`, `previousVersion?`. It has a unique index on (`organizationId`, `pluginId`).
- **Table `plugin_package`**: content-addressed by `sha256`. It stores the validated archive and its parsed manifest, and an immutable row is kept per installed version. Rolling back means re-pointing to a stored package. Nothing is fetched again, so a URL that has since disappeared cannot break a rollback.
- **Upgrade** validates the new package fully before anything changes, then swaps the active version in one transaction. **Rollback** swaps back to `previousVersion`.
- **Effect on running projects:** none. A project that was installed from a plugin app keeps a copy of the template it was installed with, just as catalog apps do today. Upgrading the plugin only changes what *new* installs and explicit "update app" actions use.
- **Uninstall** removes the contribution from the catalog view. Existing projects are left running and become ordinary services projects. Package bytes are kept until garbage collection, with no project referencing them and a grace period.
- **`GET /plugins/lock`** exports `plugins.lock.json` (`id → version → sha256 → source`), and `POST /plugins/lock` applies one. That covers reproducible installs and the RFC's "lockfile pin" requirement without a file on disk.

### D5: Trust tiers

| Tier | How it's earned | UX |
| --- | --- | --- |
| `unverified` | Default for any upload or URL. | Only an org admin can install it, after an explicit confirmation that names the images it will run. It never auto-updates. |
| `signed` | A detached ed25519 signature over the archive bytes, made by a publisher key that the **org has chosen to trust**. | Shows the publisher's name and key fingerprint. Auto-update is opt-in per plugin. |
| `verified` | Signed by an Openship key in a baked keyring, using the same pattern and rotation story as `CATALOG_PUBKEYS`. It is a separate keyring, so the module-signing key is never reused. | "Verified" badge. |

Signatures and hashes are checked fail-closed, using the same rules as `verify.ts`. Any dev escape hatch must be ignored when `NODE_ENV=production`.

### D6: Compatibility and failure isolation

- `engines.openship` is checked at install. Out of range means the install is refused, with the reason.
- After an Openship upgrade, plugins are re-checked at boot. An out-of-range plugin is marked `incompatible`, and its apps disappear from the catalog view with a notice. The host still boots.
- Re-validating a stored package that fails under a newer schema has the same effect. One bad plugin can never fail catalog loading for other plugins or for the built-in catalog.

### D7: Sources and registry

- **v1:** zip upload, and an HTTPS URL plus an expected sha256. The download is refused if the hash differs, and the download is capped in size and time.
- **Later (Phase 3):** a registry is a **static, signed index** (`index.json`: id → versions → url, sha256, signature). Openship ships with the official index configured. An org admin can add community indexes, and those default to `unverified` handling. A static index needs no hosted service to start with, and it can be mirrored.
- Git+sha and OCI sources are deferred.

### D8: Relationship to the app catalog and custom apps

- The curated catalog is unchanged. It stays reviewed, in-tree and built-in.
- Plugins are how out-of-tree authors ship apps. Built-in ids always win, and plugin app ids are namespaced (see Package format).
- Custom apps keep their current API. A later, optional migration can present each custom app as an implicit local plugin `custom.<org>/<appId>` with no version history. This ADR does not require it.

### D9: Entry points

1. **Phase 2:** REST API + `packages/contracts` schemas + SDK methods (`plugins.install|list|upgrade|rollback|disable|remove|lock`).
2. **Phase 3:** a dashboard "Plugins" section under Apps, showing install steps with the existing install-phase stepper; then the CLI (`openship plugin …`).
3. **Phase 6:** author tooling: `openship plugin pack`, `openship plugin validate`, and a template repository.

## Phasing

Each phase is its own issue and PR, and depends on this ADR being accepted.

| Phase | Deliverable | Done when |
| --- | --- | --- |
| 0 | This ADR | Maintainers accept it, or amend it |
| 1 | `plugin.json` validator in `@repo/core`, a published `plugin.schema.json`, and drift tests between the two, same as the app catalog | Fixture plugins (valid and invalid) pass and fail as expected in CI |
| 2 | Archive validator + `plugin_install` / `plugin_package` + API + SDK: install, upgrade, rollback, disable and remove for declarative plugins, upload and URL sources, `unverified` tier only | A fixture plugin built in a separate repo installs, upgrades, rolls back and uninstalls through the API |
| 3 | Dashboard + CLI, `signed` / `verified` tiers, static signed index | An operator installs a signed plugin from an index without a core PR |
| 4 | (Optional) Kind B capability packs | Separate RFC, only with a product need |
| 5 | (Optional) Kind C code plugins | Separate RFC: out-of-process isolation, capability-scoped RPC, no access to core secrets |
| 6 | Author DX | `pack` / `validate`, template repo, contract fixtures run on every core PR |

## Test bar (Phases 1 and 2)

Tests are written before the code they cover:

- manifest schema
- `manifestVersion` / `engines` gating
- namespacing and shadowing
- every archive-safety rule above, one fixture per rule
- sha256 mismatch
- trust-field stripping
- upgrade and rollback atomicity
- incompatible-at-boot deactivation
- one broken plugin not affecting the catalog

The RFC's mutation-testing and dead-code gates are worth adopting for the plugin host package once it exists. Picking the tool can wait until Phase 2.

## Answers to the RFC's open questions

1. **Is Openship the right surface?** Yes, but only for declarative plugins for now. The value is apps from outside the tree. A general code host is not justified yet.
2. **Kind A first?** Yes. It is the only kind in v1.
3. **v1 contribution points?** Only `apps`. `notifications.channel` is a declarative candidate for v1.1.
4. **Registry?** v1 has none: upload and URL with a pinned hash. Later, a static signed index, with the official index by default and community indexes added by admins and treated as `unverified`.
5. **Signing?** Optional (`signed`, with keys each org chooses to trust). It is required for the `verified` badge, which uses a separate Openship keyring built the same way as the module catalog's.
6. **UI isolation?** v1 adds no plugin UI. Dashboard panels are first-party only until a separate RFC covers iframe, CSP and postMessage.
7. **Relationship to the app catalog?** Plugins wrap apps. The catalog is unchanged, and custom apps keep working (D8).
8. **CLI, API or dashboard?** API and SDK first, then the dashboard, then the CLI.

## Alternatives considered

- **Code plugins first (VS Code style):** rejected for v1. It makes the API process part of the attack surface before the packaging, trust and versioning pipeline has been proven on data.
- **A lockfile on disk:** rejected. Openship is multi-org and stores its state in Postgres. A file would disagree with the database the first time two API instances run. The database is the source of truth, and the file is an export format.
- **Extend custom apps with a version field and nothing else:** rejected. It gives no multi-app packages, no integrity pin, no rollback storage and no trust tiers, and the next requirement would force a second migration anyway.
- **A hosted registry service in v1:** deferred. A static signed index gives the same guarantees without a new service to run.

## Consequences

- Two new tables and one new validator. No change to the deploy engine: plugin apps install through the same services pipeline as catalog and custom apps.
- App-template schema changes now affect external authors. `schemaVersion` / `minEngine` already handle this, and `engines` adds a gate at the package level.
- Kind B and Kind C stay open but are explicitly deferred, so nobody builds toward them by accident.
