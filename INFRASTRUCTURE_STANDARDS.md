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
