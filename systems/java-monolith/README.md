## App Folder — Git Submodule

The `app/` folder inside this system is **not** regular source code committed here.
It is a **Git submodule** — a pointer to the [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) repository, which holds the application's source code independently.

> This is intentional. Application code lives in its own repo. This repo focuses on deployment, infrastructure, and operations around that code.

---

### First-Time Setup

When you clone this repo, the `app/` folder will be **empty** unless you initialize the submodule:

```bash
# Option 1 — Clone with submodules from the start (recommended)
git clone --recurse-submodules https://github.com/ibtisam-iq/platform-engineering-systems.git

# Option 2 — Already cloned? Initialize manually
git submodule update --init --recursive
```

---

### Keeping `app/` Up to Date

The `app/` folder tracks a **specific commit** of the source repo, not the latest branch tip.
When the source repo (`java-monolith-app`) receives new commits, you need to explicitly pull them in:

```bash
# Step 1 — Go into the submodule folder
cd systems/java-monolith/app

# Step 2 — Pull the latest changes from the source repo
git pull origin main

# Step 3 — Go back to the root of this repo
cd ../../../

# Step 4 — Stage and commit the updated pointer
git add systems/java-monolith/app
git commit -m "chore: update java-monolith submodule to latest"
git push
```

Or in a single command:

```bash
git submodule update --remote --merge
```

> **Why do you need to commit after pulling inside the submodule?**
> The parent repo (`platform-engineering-systems`) stores only a commit SHA reference to `app/`, not the actual files.
> After you pull inside the submodule, that SHA changes locally. You need to push that updated SHA back to the parent repo so others (and CI/CD) get the same version.

---

### Important Rules

| Rule | Reason |
|---|---|
| Never add `.gitmodules` to `.gitignore` | It is the mapping file Git uses to locate the submodule repo — losing it breaks the link for everyone |
| Always clone with `--recurse-submodules` | A plain `git clone` will leave `app/` empty |
| Run `git pull` inside `app/`, not from root | Running from root with `git submodule update --remote` works too, but going inside gives you explicit control |
| Commit the updated SHA to the parent repo | Without this, the parent still points to the old commit |

---

## Build Context

Build context is set to `java-monolith/` (the parent of both `app/` and `docker/`).

`.env` lives inside the `docker/` folder at the same level as `compose.yml`.

```bash
# Run from repo root
docker compose -f systems/java-monolith/docker/compose.yml up --build

# Or from the docker folder directly
cd systems/java-monolith/docker
docker compose up --build
```

---

## Why Docker Compose Created the DB Automatically

When you use `image: mysql:8`, the official MySQL image has a built-in entrypoint script that reads specific environment variables and **automatically creates the database on first startup**:

```env
MYSQL_DATABASE=IbtisamIQbankappdb   # this line did the magic
MYSQL_ROOT_PASSWORD=your_root_password_here
```

The `MYSQL_DATABASE` variable is an **official MySQL Docker image feature** — it tells the container's init script to run `CREATE DATABASE IbtisamIQbankappdb;` automatically before MySQL even becomes available.

---

## Local MySQL Has No Such Magic

When you install `mysql-server` on Ubuntu with `apt`, it's a plain MySQL installation — no entrypoint script, no auto-database creation from environment variables. It only reads `MYSQL_DATABASE` if you're running inside the official Docker image.

| | Docker Compose | Local (`apt install`) |
|---|---|---|
| Database created by | Official MySQL image entrypoint | **You, manually** |
| Triggered by | `MYSQL_DATABASE=...` in `.env` | `CREATE DATABASE ...;` SQL |
| Tables created by | Hibernate (`ddl-auto=update`) | Hibernate (same) |
| Credentials set by | `MYSQL_ROOT_PASSWORD` in `.env` | `ALTER USER ...` SQL |

### Fix for Local — One Command

```bash
sudo mysql -u root -pIbtisamIQ -e "CREATE DATABASE IbtisamIQbankappdb;"
```

Then run your app normally:

```bash
set -a && source .env && set +a && java -jar target/bankapp-0.0.1-SNAPSHOT.jar
```

Hibernate will then take over and create all the tables automatically just like it did in Docker.
