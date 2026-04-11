
## Build Context

build context sets to `java-monolith/` (the parent of both `app/` and `docker/`)

`.env` is add to at same docker compose level (`docker`)

```bash
docker compose -f systems/java-monolith/docker/compose.yml up
```

## Why Docker Compose Created the DB Automatically

When you use `image: mysql:8`, the official MySQL image has a built-in entrypoint script that reads specific environment variables and **automatically creates the database on first startup** :

```env
MYSQL_DATABASE=IbtisamIQbankappdb   ← this line in your .env did the magic
MYSQL_ROOT_PASSWORD=your_root_password_here
```

The `MYSQL_DATABASE` variable is an **official MySQL Docker image feature** — it tells the container's init script to run `CREATE DATABASE IbtisamIQbankappdb;` automatically before MySQL even becomes available.

## Local MySQL Has No Such Magic

When you install `mysql-server` on Ubuntu with `apt`, it's a plain MySQL installation — no entrypoint script, no auto-database creation from environment variables. It only reads `MYSQL_DATABASE` if you're running inside the official Docker image.

## Side-by-Side Comparison

| | Docker Compose | Local (`apt install`) |
|---|---|---|
| Database created by | Official MySQL image entrypoint | **You, manually** |
| Triggered by | `MYSQL_DATABASE=...` in `.env` | `CREATE DATABASE ...;` SQL |
| Tables created by | Hibernate (`ddl-auto=update`) | Hibernate (same) |
| Credentials set by | `MYSQL_ROOT_PASSWORD` in `.env` | `ALTER USER ...` SQL |

## Fix for Local — One Command

```bash
sudo mysql -u root -pIbtisamIQ -e "CREATE DATABASE IbtisamIQbankappdb;"
```

Then run your app normally:
```bash
set -a && source .env && set +a && java -jar target/bankapp-0.0.1-SNAPSHOT.jar
```

Hibernate will then take over and create all the tables automatically just like it did in Docker.
