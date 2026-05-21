# Read the Docs build fix — design

**Date:** 2026-05-21
**Status:** Approved
**Scope:** Restore Read the Docs builds for `skywater-pdk` and `mithro-skywater` after the
repo move from `google/skywater-pdk` to `fossi-foundation/skywater-pdk`.

## Problem

Both Read the Docs projects associated with this repo are stuck on stale builds:

| Project | URL | Last successful build | Commit | Commits behind `main` |
|---|---|---|---|---|
| `skywater-pdk` | https://skywater-pdk.readthedocs.io/en/main/ | 2023-05-29 | `7198cf6` | 0 (at HEAD) |
| `mithro-skywater` | https://mithro-skywater.readthedocs.io/en/latest/ | 2020-11-17 | `d8e2cf1b` | 95 |

The cached HTML still serves, so the sites appear "up", but neither project has
successfully rebuilt in years. A rebuild on the current Read the Docs platform
would fail because the repo's `.readthedocs.yml` predates several mandatory
schema requirements.

## Root cause

Three concrete defects in the repo config make new builds fail under today's
Read the Docs platform:

1. **`.readthedocs.yml` is missing the `build:` block.** Read the Docs marks
   `build.os` and `build.tools` as required fields. A config without them is
   rejected at the schema-validation stage before any Sphinx work begins.

2. **`docs/environment.yml` pins `python=3.8`.** Python 3.8 is not in Read the
   Docs' supported list (`3.6`–`3.14`). The conda solve either errors or is
   silently replaced.

3. **`docs/environment.yml` references the `symbiflow` conda channel.** The
   fix-conda-channels PR #421 (commit `72f8021`) updated the root
   `environment.yml` to `litex-hub` but did not touch `docs/environment.yml`.
   The `symbiflow` channel still resolves, but it has drifted from the
   maintained channel used elsewhere in the repo.

A fourth, lower-priority issue: `docs/conf.py` hardcodes
`github_user: "google"` and `github_url: 'https://github.com/google/skywater-pdk'`,
which produces incorrect "Edit on GitHub" links after the move to
`fossi-foundation/skywater-pdk`. GitHub redirects mask the breakage but it
should be corrected.

## Out of scope

- The 114 pre-existing reStructuredText warnings (malformed tables, malformed
  cell-name roles, duplicate labels). The build exits 0 despite them; they
  predate the move.
- Switching from conda to pip-only on Read the Docs. Conda is preserved to
  minimise diff and keep local-build symmetry. A follow-up could drop conda if
  `netlistsvg` turns out not to be needed at HTML build time.
- Activating a `latest` version alias on the official project — this is a
  Read the Docs admin UI setting, not a repo change.
- The `git+https://github.com/SymbiFlow/sphinx_symbiflow_theme.git` URL in
  `docs/requirements.txt`. GitHub's 301 redirect to `f4pga/sphinx_f4pga_theme`
  still works with pip; rewriting it now is risk without benefit.

## Design

### Repo changes

Three files, three small commits.

**`.readthedocs.yml`** — add the required `build:` block. With `conda:`,
`build.tools.python` selects the conda variant (mamba vs miniconda), not the
Python version itself (which still comes from `environment.yml`):

```yaml
build:
  os: ubuntu-22.04
  tools:
    python: "mamba-latest"
```

**`docs/environment.yml`** — two edits:

- `- python=3.8` → `- python=3.11`
- `channels: [symbiflow, defaults]` → `channels: [litex-hub, defaults]`

**`docs/conf.py`** — update the GitHub references (around lines 92–97 and
172–173) from `google/skywater-pdk` to `fossi-foundation/skywater-pdk`.

### Verification

Local proof that the dependency stack still works: a fresh Python 3.11 venv
installing `docs/requirements.txt` produces a successful `sphinx-build` (exit
0, 36 MB HTML output, 114 pre-existing warnings). This is a strong proxy for
the RTD build's pip layer; the conda layer cannot be reproduced without conda
installed locally and will be validated by the actual RTD build.

End-to-end verification is the `mithro-skywater` rebuild on Read the Docs
after the branch is pushed. If that build succeeds, the same change is
PR'd against `fossi-foundation/skywater-pdk`.

### Read the Docs admin steps (out-of-band, manual)

Repo changes alone may not be sufficient — depending on what happened during
the repo move, the RTD project may also need:

- `skywater-pdk`: Admin → verify repo URL is `fossi-foundation/skywater-pdk`
  (not `google/skywater-pdk`); reinstall webhook; trigger manual rebuild.
- `mithro-skywater`: Admin → verify repo URL is `mithro/skywater-pdk`;
  reinstall webhook; trigger manual rebuild.

These are web-UI clicks; no API token is available to script them.

### Rollout

1. Branch `fix-rtd-build-config` from `main`.
2. Three commits, one per file.
3. Push branch to `mithro` remote.
4. Trigger `mithro-skywater` build on RTD; verify green.
5. Open PR against `fossi-foundation/skywater-pdk` from the same branch.
6. After merge, trigger official `skywater-pdk` rebuild.

### Rollback

All changes are reversible YAML/Python edits in version control. Existing
cached HTML on both projects continues to serve even if a new build fails —
RTD does not auto-purge passing builds.

## References

- Read the Docs v2 config schema:
  https://docs.readthedocs.com/platform/stable/config-file/v2.html
- Read the Docs supported Python/conda tooling list (in the schema docs above).
- Commit `72f8021` (root `environment.yml` channel update) — same intent,
  partial fix.
