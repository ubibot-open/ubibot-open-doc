# Deploy the Backend Server

*[中文](02-deploy-server.zh-CN.md)*

By the end of this chapter you'll have `ubibot-open-server` built and running, and you'll have
logged into the admin console for the first time.

## 1. Clone the repository

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git
cd ubibot-open-server
```

## 2. Build it

```bash
./build.sh        # Linux/macOS
# .\build.ps1      # Windows
```

This builds the admin console's frontend, then compiles the Go backend with that frontend
embedded inside it — one binary, `ubibot-server` (`ubibot-server.exe` on Windows), holding both
the API and the web UI. It'll take a minute or two the first time (downloading Go modules and npm
packages); later builds are faster.

**Screenshot placeholder:** `02-deploy-server-01.png` — the terminal output of a successful build,
ending in `==> Done: ubibot-server`.

## 3. Run it

```bash
./ubibot-server
```

Watch the log output. On a brand-new database, you'll see two lines like this:

```
no admin account found — created "admin" with a generated password: 3f9a1c...
this password is only shown once; set UBIBOT_ADMIN_PASSWORD to control it on next first run
```

> **Note:** copy that password down now. It's generated randomly and printed exactly once — there
> is no "forgot password" link. If you lose it, see the troubleshooting chapter for how to reset
> it.

The server listens on port 8080 by default (`http://localhost:8080` — the API and the admin
console are served from that same address; set the `UBIBOT_LISTEN_ADDR` environment variable to
change it).

**Screenshot placeholder:** `02-deploy-server-02.png` — the terminal showing the generated-password
log lines and the `ubibot API listening on :8080` line.

## 4. Log in

Open `http://localhost:8080` in your browser. You should land on a login page.

**Screenshot placeholder:** `02-deploy-server-03.png` — the login page.

Log in with:

- **Username:** `admin`
- **Password:** the one printed in step 3

**Screenshot placeholder:** `02-deploy-server-04.png` — the admin console's dashboard right after
logging in for the first time (it'll be mostly empty — no devices yet).

## Next

[First Login and a Tour of the Console](03-first-login-and-tour.md).
