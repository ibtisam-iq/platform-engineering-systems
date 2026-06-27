## STEP 1 — Add Submodule (FIRST TIME)

```bash
git submodule add https://github.com/ibtisam-iq/java-monolith-app.git systems/java-monolith/app
```

## STEP 2 — Initialize & Fetch

```bash
git submodule update --init --recursive
```

## STEP 3 — Commit It

```bash
git add .gitmodules
git add systems/java-monolith/app
git commit -m "Added java monolith submodule"
git push
```

> Never add `.gitmodules` into `.gitignore`.

---

# When Source Repo Changes (IMPORTANT)

You asked:

> “When I update code, how to reflect here?”

```bash
## Go inside submodule
cd systems/java-monolith/app
## Pull latest changes
git pull origin main
## Go back
cd ../../../
## Update parent repo
git add systems/java-monolith/app
git commit -m "Updated submodule"
git push
```

---

# Alternative (Single Command Update)

```bash
git submodule update --remote --merge
```

👉 pulls latest commits from source repo

---
