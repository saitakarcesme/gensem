# Private GitHub Repo → Coolify Deployment — GitHub App Setup

> Last verified: 2026-07-10 (documentation review; GitHub App + Coolify Source flow as implemented since GSE-One v0.64.0 — external UI labels may drift)

**Why this exists:** Coolify cannot clone a **private** GitHub repo via anonymous HTTPS — it returns `422` from the `/applications/public` endpoint. To unblock this, Coolify supports a "GitHub App" source: someone registers a GitHub App once on the Coolify instance, each repo owner installs that App on their own GitHub account with read access to their repo, and `deploy.py` targets the authenticated source via the `COOLIFY_GITHUB_APP_UUID` variable (auto-routing since v0.64.0).

**Two roles, which may be the same person:**

| Role | Who it is | What they do |
|------|-----------|--------------|
| **App owner** | Whoever operates the Coolify instance — **yourself in solo mode**, or the trainer in a training context | One-time setup: sections A + (optionally) C, D |
| **App installer** | Whoever owns the private repo to deploy — **yourself in solo mode**, or each learner | Per-repo authorization: section B |

**Solo mode:** you are both roles. Do section A once on your own Coolify + GitHub account, then follow section B's three steps yourself (you already know the two values from A.3 — no email needed).

**Training mode:** the trainer does section A once; each blocked learner receives the section B steps (a ready-to-send email template is in the Annex).

---

## A. App owner — one-time setup (~15 min)

Do this **once** per Coolify instance (reusable across all repos, installers, and future cohorts).

### A.1 — Coolify: create the GitHub App source

1. In the Coolify UI (left sidebar): **Sources → + New → GitHub App**.
2. Fill in:
   - **Name**: a reusable identifier (e.g., `<your-namespace>-deploy`).
   - **Organization (on GitHub)**: your own GitHub account or a dedicated org you own (NOT the installer's account — see Ownership note below).
   - **System Wide**: check if your Coolify instance will serve multiple teams; leave unchecked otherwise.
   - **Self-hosted / Enterprise GitHub** fields: leave at defaults (`https://github.com`, `https://api.github.com`, `git`, `22`) unless you're using GHE Server.
3. Click **Continue**, then on the next page click **Register Now** (the Automated Installation path).
4. You are redirected to `github.com` with a pre-filled manifest. On the "Create GitHub App for `<your-account>`" page, do **not** modify the permissions — click **Create GitHub App**.
5. GitHub redirects back to Coolify. Your Source is now registered (App ID, Client ID/Secret, Webhook Secret, Private Key all auto-populated).

**Ownership note.** The GitHub App is owned by the App owner — *not* by any installer. The App owner holds the private key that Coolify uses to authenticate. Each installer *installs* this App on their own GitHub account as a separate, controlled gesture; installation ≠ ownership. An installer can revoke their installation at any time without affecting anyone else.

### A.2 — **CRITICAL** — Make the App publicly installable (skip in pure solo mode)

By default, the Coolify-generated manifest creates the App in **"Only on this account"** mode. In **solo mode where your repos live on the same GitHub account that owns the App, this is fine — skip this step.** But if anyone else (learners, teammates, or a second account of yours) must install it, they will see *"This GitHub App is marked as private and cannot be installed on this account"*. This step is frequently missed and is the **most common failure point** of the multi-account setup.

Fix:

1. Open `https://github.com/settings/apps/<your-app-slug>`.
2. Click the **Advanced** tab in the left sidebar.
3. Scroll down to the **Danger zone** section (cosmetic — this action is reversible).
4. Find the block **"Make this GitHub App public"** (*"Allow this GitHub App to be installed on other accounts"*).
5. Click **Make public**. Confirm the dialog.

**Verify** via `gh` CLI (optional but recommended):

```bash
gh api /apps/<your-app-slug> --jq '.slug, .html_url, .owner.login'
```

If this returns three lines (slug, html_url, owner) with no error, the App is publicly installable.

### A.3 — Collect the two values every installer needs

1. **Install link** — `https://github.com/apps/<your-app-slug>/installations/new`
   (Navigate to `github.com/settings/apps/<your-app-slug>` → **Install App** in the left sidebar → copy the URL of the page you land on.)
2. **Coolify Source UUID** — visible in the URL of the Coolify Source page (`https://<your-coolify>/source/github/<UUID-here>`).

Both values are **identical for all installers**, forever (until you rotate or delete the App). Keep them in your notes.

### A.4 — Owner self-install (recommended smoke test)

Before relying on the setup (or exposing it to learners), install the App on your own GitHub account as a smoke test:

1. Click **Install Repositories on GitHub** on the Coolify Source page (or open the install link directly).
2. Select your own account → "Only select repositories" → tick any test repo → Install.
3. Return to the Coolify Source page, refresh. The red warning *"You must complete this step before you can use this source!"* should disappear, and an Installations row for your account should appear.

If the warning remains after a refresh, investigate (webhook delivery logs in the App's Advanced tab, or Coolify logs).

---

## B. App installer — authorize and redeploy (~5 min)

You hit a **422** during `/gse:deploy` because your GitHub repo is private. Three steps: Step 1 is a browser action (GitHub login — your IDE agent cannot do it for you); Steps 2 and 3 are **prompts you paste to your GSE orchestrator chat** (Claude Code, Cursor, opencode…), which adopts the `deploy-operator` role and runs the underlying `deploy.py` operations.

You need the two values from A.3: the **install link** and the **Source UUID**. In solo mode you produced them yourself; in a training context your trainer provides them.

### Step 1 — Install the GitHub App on your GitHub account (~2 min, browser)

1. Open the install link: `https://github.com/apps/<APP-SLUG>/installations/new`
2. GitHub asks which account to install the App on. **Choose the account that owns the repo you want to deploy.**
3. On the scope screen: do **NOT** pick "All repositories" (too broad) — pick **"Only select repositories"**.
4. **Tick only the repo you want to deploy** (the one passed to `--github-repo`).
5. Click the green **Install** button.

**Your privacy guarantees stay full:** the repo remains private, the App has **only read access** (Contents: Read, Metadata: Read, Pull requests: Read) on the repo you ticked, and **only you** can revoke that access at any time from `github.com/settings/installations`.

### Step 2 — Persist the Coolify Source UUID (~30 s)

Paste this prompt to your GSE orchestrator, at the root of your project:

> I just installed the GitHub App authorizing Coolify access to my private repo. Please persist the Coolify Source UUID `<SOURCE-UUID>` into the `COOLIFY_GITHUB_APP_UUID` variable of my project's `.env`, using the `deploy.py env-set` subcommand (or the equivalent routing for my install mode). Then show me the resulting `.env` line so I can confirm it was written correctly.

### Step 3 — Relaunch the deployment sequentially

Paste this prompt:

> Please re-run the `/gse:deploy` Phase 6 step for my application(s). My prior attempt failed with a Coolify `422` because my repo was private; I have now installed the GitHub App (Step 1) and `COOLIFY_GITHUB_APP_UUID` is set in `.env` (Step 2). Use the exact same `deploy-app` parameters you generated earlier (check our previous conversation for `--name`, `--subdomain`, `--github-repo`, `--branch`, `--type`, `--port` — or read them from `.gse/deploy.json → applications[]` if available). If I have multiple apps to deploy, run them **one at a time, sequentially** — wait for each to return a `healthy` health check AND print its final URL before starting the next. Do NOT parallelize. Report back each app's URL when it becomes healthy.

The sequential constraint matters: parallel invocations can create duplicate Coolify projects.

### If 422 still persists

Four possible causes — for each, paste the suggested prompt to your orchestrator:

1. **App marked as private** (multi-account case) — GitHub showed *"This GitHub App is marked as private and cannot be installed on this account"* at Step 1. Only the App owner can fix it (section A.2). Solo mode: do A.2 yourself; training mode: ask your trainer.
2. **App not actually installed** — prompt: *"Check whether the GitHub App `<APP-SLUG>` is installed on my GitHub account, and which repositories it has access to (`gh api /user/installations`, match `app_slug`)."* If missing → redo Step 1.
3. **Target repo not ticked during install** — prompt: *"Show me which repositories the `<APP-SLUG>` App installation has access to; if my target repo is missing, explain how to add it."* (Typically: `github.com/settings/installations` → Configure → tick the repo → Save.)
4. **Wrong UUID in `.env`** — prompt: *"Show me the current value of `COOLIFY_GITHUB_APP_UUID` in my project's `.env` (via `deploy.py env-get`)."* Compare against `<SOURCE-UUID>` — no whitespace, quotes, or URL fragments.

If none of these resolves the 422, the deployment tooling itself is end-to-end since v0.64.0 (see section E) — so a persistent 422 points to one of the causes above. In training mode, send the App owner the exact error message.

---

## C. App owner — observability while an installer is acting

As an installer executes Step 1, two things happen that the App owner can watch:

1. **GitHub → App Advanced tab → Recent Deliveries**: a new `installation.created` event with a green checkmark (`https://github.com/settings/apps/<APP-SLUG>/advanced`).
2. **Coolify → Sources → Your App**: a new Installations row for the installer's GitHub account, listing the ticked repo (`https://<your-coolify>/source/github/<SOURCE-UUID>`).

If neither appears within a few minutes, the install likely failed silently — ask for a screenshot of the GitHub page right after the Install click. Once Step 3 runs, new **Resources** rows appear under the same Source page (one per deployed app), with live build logs.

---

## D. Reuse for subsequent installers

For every new installer after the first, the App owner does **nothing** in Coolify or GitHub — the same install link and Source UUID work for everyone. A single Coolify Source supports N installations (one per GitHub account), with no limit. Per-cohort App isolation is possible (one App per cohort) but usually unnecessary: each installation is already repo-scoped and independently revocable.

---

## E. Implementation status — deploy.py auto-detection

Since GSE-One v0.64.0, `deploy.py` dispatches to the Coolify `/applications/private-github-app` endpoint automatically when `COOLIFY_GITHUB_APP_UUID` is set (implemented as `create_private_github_app_application` in `coolify_client.py` + routing in `deploy_app()`, with `server_uuid` auto-resolution). Once the App is installed and the UUID is set, re-running `deploy-app` completes the deployment with no manual Coolify UI step.

---

## Annex — Training context: ready-to-send email template

Trainers: when a learner hits the 422, send them this email with the **two placeholders** (`<APP-SLUG>`, `<SOURCE-UUID>`) replaced by your values from A.3. The body mirrors section B.

**Subject:** Coolify deployment — private GitHub repo: 3 steps on your side (~5 min)

Hi `<first-name>`,

You ran `/gse:deploy` and the GSE agent returned a **422** error from Coolify, flagging your **private GitHub repo** as the blocker. This is expected: Coolify cannot clone a private repo without explicit authorization from you. You have **3 small steps** left — Step 1 is a quick browser click, Steps 2 and 3 are prompts to paste into your GSE orchestrator chat. Follow section B of the guide `references/private-repo-github-app-setup.md` shipped with your GSE-One install, using these values:

- Install link: `https://github.com/apps/<APP-SLUG>/installations/new`
- Coolify Source UUID: `<SOURCE-UUID>`

Happy deploying!

Your Trainer

**Trainer tip:** set `COOLIFY_GITHUB_APP_UUID` (and `SERVER_UUID` if several servers are visible) in your own `.env` **before** running `/gse:deploy --training-init` — both keys are then embedded in the distributed `.env.training`, and learners skip Step 2 entirely.
