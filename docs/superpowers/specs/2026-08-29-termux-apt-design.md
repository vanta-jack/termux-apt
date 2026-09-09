# termux-apt Design Spec

**Date:** 2026-08-29
**Scope:** `vanta-jack/termux-apt` — central APT repository serving all `vanta-jack` Termux packages via GitHub Pages.

---

## Context

A single, personal third-party APT repository for native Termux aarch64 packages built across multiple project repos (termux-litellm, future doppler-cli, tailscale, etc.). Bootstrapped once per fresh Termux install; maintained automatically via CI dispatches from each project repo.

---

## Architecture

```
vanta-jack/termux-litellm  ─── workflow_dispatch (version, deb_url, sha256) ──►┐
vanta-jack/doppler-cli     ─── workflow_dispatch ──────────────────────────────►│
vanta-jack/tailscale       ─── workflow_dispatch ──────────────────────────────►│
                                                                                 ▼
                                                              vanta-jack/termux-apt
                                                              update-package.yml
                                                                │
                                                      GitHub Environment "production"
                                                         manual approval (you)
                                                                │
                                                      download + verify SHA-256
                                                      overwrite pool/<package>_*.deb
                                                      run termux-apt-repo
                                                      deploy to gh-pages
                                                                │
                                               vanta-jack.github.io/termux-apt/
                                                                │
                                              pkg install litellm / pkg upgrade litellm
```

---

## Repo Structure

```
termux-apt/
├── pool/
│   └── litellm_X.X.X_aarch64.deb      # current version only — overwritten on each release
├── dists/
│   └── stable/
│       └── main/
│           └── binary-aarch64/
│               ├── Packages
│               ├── Packages.gz
│               └── Release
├── setup.sh                             # one-liner bootstrap for fresh Termux installs
└── .github/workflows/
    └── update-package.yml               # receives dispatch → approval gate → updates pool → deploys Pages
```

---

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Hosting | GitHub Pages (`gh-pages` branch) | Free, HTTPS, stable URL, no quota concern for metadata-only |
| `.deb` in pool | Current version only, overwritten each release | Manages Pages storage; historical versions preserved in project GitHub Releases |
| Signing | `[trusted=yes]`, no GPG | Single-user homelab; HTTPS + SHA-256 verification covers transport integrity |
| Approval gate | GitHub Environment `production` with required reviewer | Manual gate before any package lands in production APT index |
| Naming | `termux-apt` | Avoids collision with official `termux/termux-packages`; not confusable with TUR |
| Multi-package | One `pool/`, one index, all packages | Standard pattern (ref: TUR); single `sources.list` entry covers all packages forever |
| APT tool | `termux-apt-repo` | Standard Termux community tool; generates correct `dists/` structure |

---

## Bootstrap (Fresh Termux Install)

```bash
curl -fsSL https://vanta-jack.github.io/termux-apt/setup.sh | bash
pkg update
pkg install litellm
```

`setup.sh` writes one file:
```
$PREFIX/etc/apt/sources.list.d/vanta-jack.list
  → deb [trusted=yes] https://vanta-jack.github.io/termux-apt/ stable main
```

---

## CI Workflow: `update-package.yml`

Triggered by `workflow_dispatch` from any project repo. Dispatch payload:
```json
{ "package": "litellm", "version": "1.x.x", "deb_url": "https://...", "sha256": "abc123..." }
```

Steps:
1. **GitHub Environment `production` gate** — requires your manual approval in GitHub UI before any step runs
2. Download `.deb` from `deb_url`
3. Verify SHA-256 matches payload — fail loudly if mismatch, pool untouched
4. Overwrite `pool/<package>_*.deb` with downloaded file
5. Run `termux-apt-repo pool/ .` → regenerate `dists/` index
6. Commit updated `pool/` + `dists/` to `gh-pages` branch
7. GitHub Pages auto-deploys

---

## Error Handling

| Failure | Behaviour |
|---|---|
| SHA-256 mismatch on download | Workflow fails; pool and index untouched |
| `termux-apt-repo` fails | Workflow fails; gh-pages deploy skipped |
| GitHub Pages deploy fails | Pool updated but index not live; re-run workflow |

---

## Adding Future Packages

No structural changes to `termux-apt` needed. Each new project repo:
1. Builds its `.deb`
2. Publishes to its own GitHub Releases
3. Fires `workflow_dispatch` to `termux-apt` with its payload

`termux-apt-repo` automatically includes all `.deb` files in `pool/` in the generated index. After adding a new package, users install with `pkg install <new-package>` — no `setup.sh` re-run required.

---

## Rollback

`termux-apt` always serves only the current version. Rollback to a prior version is done directly from the project repo's GitHub Releases:

```bash
curl -L https://github.com/vanta-jack/termux-litellm/releases/download/vX.X.X/litellm_X.X.X_aarch64.deb -O
dpkg -i litellm_X.X.X_aarch64.deb
apt-mark hold litellm      # optional: prevent auto-upgrade while investigating
```

`pkg upgrade` will upgrade from the held version to current when unhold is run.

---

## Deferred

- **GPG signing** — add when sharing this repo beyond personal use. Steps: generate GPG keypair, add private key as GitHub Actions secret, update `update-package.yml` to sign `Release` file, update `setup.sh` to import public key. `# ponytail: [trusted=yes] sufficient for personal homelab; add GPG before any public distribution`
- **Per-package approval granularity** — currently one `production` environment gates all packages. If package count grows large, consider per-package environments. YAGNI until then.
