# Infrastructure Standards

Rules for Docker, CI/CD, and deployment configuration. Read `COMMON_CODING_STANDARDS.md` first.

---

## Include All System Dependencies in Dockerfiles

Before writing a Dockerfile, identify every system-level dependency required by the
Python packages in use (e.g. `libpq-dev` for asyncpg, `gdal-bin` + `libgdal-dev` for
GeoAlchemy2, `libmagic` for python-magic). A missing system dependency causes an
`ImportError` at container startup — not at build time — making it hard to diagnose.

---

## CI Platform — Use the Correct Config File

Check which CI/CD platform the project uses before generating pipeline config:
- **GitLab**: `.gitlab-ci.yml` — uses `services:` with alias-based networking.
- **GitHub Actions**: `.github/workflows/*.yml` — uses `services:` with `localhost`-based networking.

The syntax, service networking, and trigger syntax differ significantly. Never generate
GitHub Actions config for a GitLab project or vice versa.

---

## Follow the File Layout in the Plan Exactly

Do not create additional directories, duplicate config files, or parallel implementations
beyond what the plan specifies. If the plan says `docker-compose.prod.yml` at the repo
root, create exactly that file — not also a `deploy/docker-compose.prod.yml`. Extra files
that reference each other inconsistently introduce bugs and ambiguity about which config
is authoritative.

---

## Dependency and Image Provenance

Local/path dependencies (an editable install from a sibling repo, a vendored package)
and container base images belong in the same dependency inventory as registry packages —
see `COMMON_CODING_STANDARDS.md` § Dependency Tracking. They're easy to forget because no
package-manifest scan lists them, but they carry the same supply-chain risk.

- **Pin a local path dependency's source** to a specific revision (git SHA or tag) or
  consume it as a content-addressed artifact (a built wheel, a locked digest) for
  anything that produces a release or a deployed image. A mutable path/branch reference
  means the same app commit and lockfile can produce different runtime behavior depending
  on what the sibling checkout happened to contain at build time.
- **CI and the release pipeline must resolve that local dependency the same way.** If CI
  fetches a sibling repo's default branch while the release script points at a different
  local path (or vice versa), pin both to the same mechanism — e.g. a commit SHA recorded
  alongside the app's own version — so a release build and the CI build that validated it
  are guaranteed to have used the same sibling source.
- **Document how a clean checkout obtains it.** If any dependency comes from a local
  path, a private git repository, or an internal package index rather than the public
  registry, the project's setup docs (README, `CLAUDE.md`/`AGENTS.md`, or
  `CONTRIBUTING.md`) must say explicitly how a clean checkout obtains it — which sibling
  repo to clone, at what path, with what credentials. CI config that clones the sibling
  repo as a build step is not a substitute for this: a new contributor or a local dev
  environment follows the setup docs, not the CI YAML.

---

## Lockfile Freshness Is Not the Same as a Frozen Install

A `--frozen` (uv) / `--locked`-style install flag skips dependency resolution and installs
exactly what the lockfile says — it does not verify the lockfile is still consistent with
the manifest (`pyproject.toml`, `package.json`). A pipeline comment or job name that
implies "`--frozen` validates the lock is fresh" is wrong and will hide a stale lockfile
from review. Freshness is a separate check (e.g. `uv lock --check`, or `npm ci`, which
does fail on a `package.json`/`package-lock.json` mismatch) — run that explicitly where
freshness matters to the pipeline; don't rely on a frozen-install flag to provide it.
