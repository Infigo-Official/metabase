# Metabase Local Setup (v0.63.15)

How to build and run the Infigo fork locally.

Everything compiles **inside the Docker build**, so you do not need Node, Java or Clojure on your
machine — only Docker. The versions the build uses (Node 22, Temurin 25 JDK, Clojure 1.12) are
pinned in the `Dockerfile`.

---

## Prerequisites

| Tool | Minimum | Purpose |
|------|---------|---------|
| Docker Desktop | 20.10 | Build and run |
| Docker Compose | 2.0 | Orchestrate the stack |
| Git | any | Clone |

Give Docker plenty of headroom — the frontend build plus the Clojure uberjar is heavy. **8 GB RAM
and ~20 GB free disk**, and expect **15–30 minutes** for a cold build. Later builds reuse layers and
are much faster as long as `package.json` / `deps.edn` are untouched.

---

## 1. Clone and check out the branch

```bash
git clone https://github.com/Infigo-Official/metabase.git
cd metabase
git checkout infigo_v0_63_15
```

### Line endings

Nothing to do. `.gitattributes` pins `*.sh` to `eol=lf`, so shell scripts stay LF even with
`core.autocrlf=true` on Windows.

This used to be a manual step. Without it the build fails with:

```
/usr/bin/env: 'bash\r': No such file or directory
```

because `COPY . .` puts CRLF scripts into a Linux image. If you ever see that error, your checkout
predates the `.gitattributes` fix — renormalize:

```bash
git ls-files -z '*.sh' | xargs -0 rm -f
git ls-files -z '*.sh' | xargs -0 git checkout --
```

---

## 2. Build and run

From the repo root:

```bash
docker compose up -d --build
```

That builds `infigo-metabase:v0.63.15` from this repo and starts it against a throwaway Postgres 17.
Metabase comes up on <http://localhost:3000>; the database is exposed on host port `55434` so it
cannot collide with other local Postgres containers.

Watch it come up:

```bash
docker compose logs -f metabase
```

First boot runs the full application-database migration set and takes a few minutes.

To build without running:

```bash
docker build -t infigo-metabase:v0.63.15 \
  --build-arg MB_EDITION=oss \
  --build-arg VERSION=v0.63.15 .
```

> `.dockerignore` excludes `.git`, so the build cannot read the version from git history. Always
> pass `VERSION` explicitly — `docker-compose.yml` already does.

Tear down, keeping data:

```bash
docker compose down
```

Tear down and **delete** the database:

```bash
docker compose down -v
```

---

## 3. Running against the richer local stack

If you already have a fuller local environment (SQL Server, a reporting Postgres, pgAdmin), build
the image here and point that stack at the tag instead of the published one:

```yaml
services:
  metabase:
    image: infigo-metabase:v0.63.15   # was metabase/metabase:v0.50.26
```

### Before you do that — read this

An existing Metabase Postgres volume from an older version holds a **v0.50.26 application
database**. Starting v0.63.15 against it runs the full 0.50 → 0.63 migration set, which is
**one-way**. Metabase cannot downgrade an application database, so the old instance — its
questions, dashboards, users and permissions — is gone the moment the new container boots. It is
not recoverable by switching the image tag back.

Keep the old instance by copying the volume first and running the new version against the copy:

```bash
docker run --rm \
  -v <old_volume>:/from -v metabase_063_pgdata:/to \
  alpine sh -c "cd /from && cp -a . /to"
```

Then point the new stack's `db` service at `metabase_063_pgdata`.

Doing this deliberately, on a **restored snapshot of a real application database**, is the single
most useful pre-upgrade test available: it proves the migration path end to end and tells you how
long the production maintenance window needs to be.

---

## 4. Verify the fork's change is live

```bash
git diff --stat v0.63.15..infigo_v0_63_15
# src/metabase/users_rest/api.clj | 13 +++----------
```

Then, against the running instance, as a user who is **not** an admin and **not** sandboxed:

```
GET /api/user/recipients
```

The response must contain only users sharing at least one group with the caller. On stock upstream
OSS it returns every active user — that difference is the whole point of the fork. See `INFIGO.md`.

---

*All other drivers, versions, and unrelated code are not relevant to this setup.*
