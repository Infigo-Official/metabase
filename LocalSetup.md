# Metabase Local Setup (v0.50.26)

These instructions walk you through cloning Metabase, building the **v0.50.26** image, and running it locally with Docker Compose.

---

## Prerequisites

| Tool               | Minimum Version | Purpose                                    |
|--------------------| --------------- | ------------------------------------------ |
| **Docker**         | 20.10           | Build & run containers                     |
| **Docker Compose** | 2.0             | Orchestrate the Metabase stack             |
| **Node.js**        | 16              | Build frontend assets & local dev tasks    |
| **Git**            | any             | Clone the repository & manage line‑endings |

> **Tip** On Windows or WSL 2, ensure Docker Desktop is running before you begin.

---
---

## 1 Clone the repository & check out the release tag

```bash
# 1. Clone the Metabase repo
 git clone https://github.com/metabase/metabase.git
 cd metabase

# 2. Check out the required tag
 git checkout v0.50.26
```

---

## 2 Normalise line endings (recommended on Windows)

Metabase’s build scripts expect Unix‑style line endings. Configure your local clone to ***input*** line endings so Git converts CRLF↔LF automatically:

```bash
# Set automatic line ending conversion to "input" for this repo only
 git config --local core.autocrlf input

# Verify the setting
 git config core.autocrlf   # should output: input
```

On top of that create a file in the solution directory .gitattributes and add the following contents to it:

```gitattributes
*.sh text eol=lf
```

---

## 3 Build the Metabase Docker image

Inside the repository root run:

Build the OSS edition of Metabase v0.50.26

You can run just docker compose or do it manually using this command. Otherwise just skip to the next step

```bash
   docker build -t metabase:v0.50.26 --build-arg MB_EDITION=oss --build-arg VERSION=local-$(git rev-parse --short HEAD) .
```

🕒 The build typically completes in **5–10 minutes** (depending on network speed & CPU).

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

```yaml
version: "3.9"

services:
  metabase:
    image: metabase:v0.50.26
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
    image: postgres:16
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

---

🎉 You now have Metabase **v0.50.26** running locally. Happy exploring!
