# Metabase Local Setup (v0.63.15)

These instructions walk you through cloning Metabase, building the **v0.63.15** image, and running it locally with Docker Compose.

> **NOTE!** Locally this won't run properly unless you have adjusted the crlf line endings that conform to the windows machine and there might be issues with that during the docker build image stage. This is why the test of all of this was done in Ubuntu VM that is inherently supports Linux and lf issues are non existent.

> **What it looks like when you get this wrong:** the build *succeeds*, every asset serves HTTP 200, `/api/health` returns 200 — and the page is blank, with no error anywhere. Webpack compiles the CRLF sources into an `app-main.js` that never mounts. The quickest tell is comparing bundle hashes with an official image of the same version: `vendor.js` will match (it comes from `bun install`) while `app-main.js` will not.


---

## Prerequisites

| Tool               | Minimum Version | Purpose                                    |
|--------------------| --------------- | ------------------------------------------ |
| **Docker**         | 20.10           | Build & run containers                     |
| **Docker Compose** | 2.0             | Orchestrate the Metabase stack             |
| **Git**            | any             | Clone the repository & manage line‑endings |

> **Tip** On Windows or WSL 2, ensure Docker Desktop is running before you begin.

---
---

## 1 Clone the repository & check out the release tag

```bash
# 1. Clone the Metabase repo
 git clone https://github.com/Infigo-Official/metabase.git
 cd metabase

# 2. Check out the Infigo fork branch
 git checkout infigo_v0_63_15
```

---

## 2 Normalise line endings (recommended on Windows)

Metabase’s build scripts expect Unix‑style line endings. Configure your local clone to ***input*** line endings so Git converts CRLF↔LF automatically:

> **NOTE!** Do not push these changes on the github as this will mess the repository. This should only be done for local setup.

```bash
# Set automatic line ending conversion to "input" for this repo only
 git config --local core.autocrlf input

# Verify the setting
 git config core.autocrlf   # should output: input
```

> **IMPORTANT!** Setting the config does **not** rewrite files that are already on disk. If you cloned before setting it, re-checkout the working tree so the existing files are rewritten as LF:

```bash
git ls-files -z | xargs -0 rm -f
git checkout -- .
```

Verify before building — neither file may report `CRLF line terminators`:

```bash
file bin/build.sh frontend/src/metabase/app-main.js
```

Cloning inside a Linux VM or WSL avoids all of this: a native Linux checkout is LF by construction, which is why the image is built there.

---

## 3 Build the Metabase Docker image

Inside the repository root run:

Build the OSS edition of Metabase v0.63.15

You can run just docker compose or do it manually using this command. Otherwise just skip to the next step

```bash
   docker build -t infigo-metabase:v0.63.15 --build-arg MB_EDITION=oss --build-arg VERSION=local-$(git rev-parse --short HEAD) .
```

🕒 The build typically completes in **15–30 minutes** (depending on network speed & CPU).

---

## 4 Spin up Metabase with Docker Compose

### 4.1 Use an existing compose file

If your project already includes a suitable `docker-compose.yml`, simply run:

```bash
docker compose up -d   # starts Metabase in the background
```

or if you want to rebuild your changes use:

```bash
docker compose up -d --build   # rebuilds if needed, restarts stack
```

### 4.2 Create a minimal compose file (if you don’t have one)

Paste the snippet below into `docker-compose.yml` at the project root:
This is only meant for running locally.

```yaml
version: "3.9"

services:
  metabase:
    image: infigo-metabase:v0.63.15
    build:
      context: .                # repo root
      args:
        MB_EDITION: oss
    container_name: metabase-dev
    environment:
      MB_DB_TYPE: postgres
      MB_DB_HOST: db
      MB_DB_DBNAME: metabase
      MB_DB_USER: metabase
      MB_DB_PASS: metabase
      MB_JETTY_PORT: 3000
    ports:
      - "3000:3000"
    depends_on:
      - db
    volumes:
      - plugins:/plugins   # hot-drop drivers if needed

  db:
    image: postgres:17
    container_name: metabase-db
    environment:
      POSTGRES_USER: metabase
      POSTGRES_PASSWORD: metabase
      POSTGRES_DB: metabase
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
  plugins:
```

Then start the service:

```bash
docker compose up -d
```

---

## 5 Access Metabase

Once the container is healthy, open your browser at **[http://localhost:3000](http://localhost:3000)** and complete the onboarding wizard.

---

## 6 Stopping & cleaning up

```bash
# Stop containers but keep data
docker compose down

# (Optional) remove the persistent volume
# WARNING: this deletes any saved questions & configs!
docker volume rm metabase-data
```

---

## Troubleshooting

| Symptom                        | Fix                                                                                                                      |
| ------------------------------ |--------------------------------------------------------------------------------------------------------------------------|
| Image build fails behind proxy | Set `HTTP_PROXY`/`HTTPS_PROXY` env vars before running `docker build`.                                                   |
| Port **3000** already in use   | Change the left‑hand port in `ports:` (e.g. `- "4000:3000"`) & browse to [http://localhost:4000](http://localhost:4000). |
| Container exits immediately    | Check logs with `docker compose logs -f` for errors such as incorrect Java version.                                      |
| Page loads blank, no console error | CRLF line endings leaked into the build. See §2 — the build succeeds and serves 200s regardless. Re-checkout as LF and rebuild. |

---

🎉 You now have Metabase **v0.63.15** running locally. Happy exploring!
