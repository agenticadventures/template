# Isolated Claude Code Agent — Complete Setup Guide

A step-by-step runbook for running **fully autonomous, never-prompted** Claude Code agents
(`--dangerously-skip-permissions` / bypass mode) in a completely isolated environment.

**What this setup guarantees**

- The agent runs on a throwaway cloud VM (GitHub Codespaces) — **never on your computer**.
- The agent can reach **only one repository** and **the open internet** — nothing else you own.
- Both costs are **hard-capped**: Claude token spend and cloud compute.
- Permission prompts are turned **fully off** — so the *environment* is the safety boundary,
  not the prompts. That's the whole point of the isolation.

**The pieces**

| Piece | Role |
|---|---|
| New Gmail account | Clean identity for the GitHub *and* Anthropic accounts |
| New GitHub account | Account-level isolation (no other repos/orgs to reach) |
| New Anthropic account + capped key | Disposable, budget-bounded API credential ($20/mo cap) |
| One public repo | Where the agent works; also enables free GitHub Pages |
| GitHub Codespaces | Cloud VM the agent runs on (not your machine) |
| GitHub Pages | Publishes whatever the agent builds |
| Bypass config (alias + settings) | Makes every session run with no permission prompts |

Estimated time: ~30–45 minutes, most of it one-time account creation.

---

## Phase 1 — New Gmail account

This becomes the identity (and recovery email) for the GitHub **and** Anthropic accounts.

1. Open a fresh browser profile or incognito window so it doesn't entangle with your existing
   Google login.
2. Go to <https://accounts.google.com> → **Create account** → **For my personal use**.
3. Enter a name (a pseudonym/agent name is fine), choose the username (this is the email you'll
   use everywhere below), and set a strong password. Save it in your password manager.
4. Complete phone verification if prompted — Google usually requires it for new accounts. The
   same number can verify a small number of accounts. A recovery email is optional.
5. Turn on 2-Step Verification: **Google Account → Security → 2-Step Verification**. Prefer an
   authenticator app over SMS.

---

## Phase 2 — Dedicated Anthropic account + capped key

Use a **separate** Anthropic Console account on the new Gmail — *not* your existing account or Max
plan. This keeps the billing identity disposable too, so the only credential the agent ever holds
is a workspace-scoped, hard-capped API key, never anything tied to your real account. (A
subscription OAuth token would authenticate the whole account, has no hard dollar cap, and is the
usage pattern that gets flagged for autonomous agents — wrong tool for this job.)

6. Sign up for a **new account** at <https://console.anthropic.com> using the **new Gmail
   address**. This creates a brand-new organization, fully separate from any existing Anthropic
   account or subscription. Complete email/phone verification.
7. Add a payment method under **Settings → Billing**. A key won't authorize without billing on
   file, even under a cap — the cap limits spend, it doesn't replace billing.
8. Set the organization's **monthly spend limit to $20** under **Settings → Billing → Limits**.
   Because this org is single-purpose, this is your hard ceiling: when monthly spend reaches $20,
   keys stop authorizing requests — a hard stop, not just an alert. You can raise it later if a
   run needs more.
9. Create a **Workspace** (e.g. `isolated-agent`) under **Settings → Workspaces**. A key created
   in a workspace can only access resources within that workspace.
10. With the `isolated-agent` workspace selected (top-left selector), create an **API key inside
    it** and copy it — it's shown only once. Don't create the key in the **Default** workspace;
    Default can't take its own spend limit.

> Keep this key handy for Phase 4. Treat it as the only sensitive thing the agent can touch — on a
> throwaway account, behind a $20 ceiling. That's what makes it safe to hand to a never-prompted
> agent.

---

## Phase 3 — New GitHub account

11. Sign up at <https://github.com> using the new Gmail address. Choose a username that signals
    automation (it helps frame this as a *machine account*, which GitHub's terms explicitly permit
    alongside your personal account).
12. Verify the email, complete any phone check, and **enable 2FA** (GitHub requires it for most
    accounts).
13. Decide your compute ceiling:
    - **Hard free cap:** add no payment method. Codespaces simply stops when the free monthly
      hours are exhausted (~60 hours of 2-core runtime).
    - **Keep-it-running:** add a card, then set a low limit under **Billing → Budgets** with
      "pause at limit."

---

## Phase 4 — Repository + secret

14. Create **one public repository** and initialize it with a README. Public is required because
    free personal accounts can only publish GitHub Pages from public repositories. This is fine
    for the threat model: a Pages site is world-viewable anyway, and no secrets live in the repo.
15. Add the Anthropic key as a **repository-level Codespaces secret**: repo → **Settings →
    Secrets and variables → Codespaces → New repository secret**.
    - Name it exactly: `ANTHROPIC_API_KEY`
    - Value: the workspace key from Phase 2.

    Scoped to this repo, it's automatically injected as an environment variable inside the
    Codespace, and Claude Code reads it on launch.

---

## Phase 5 — Add the scaffold files (preconfigures bypass mode)

Drop the files from this package into the repo. They make the Codespace come up agent-ready and
fully unprompted. 

File structure:

.devcontainer/
  devcontainer.json
.claude/
  settings.json
.github/
  workflows/
    pages.yml
index.html
README.md

- **`.devcontainer/devcontainer.json`** — installs Claude Code on a Node 20 image (the npm
  package needs Node 18+), aliases `claude` to always carry `--dangerously-skip-permissions`,
  and persists `~/.claude` across rebuilds so you don't re-accept the one-time warning.
- **`.claude/settings.json`** — sets `permissions.defaultMode: bypassPermissions`. This is the
  documented "start every session unprompted" setting. The alias is the reliable enforcer because
  the settings-only default has been buggy across versions, so both are included.
- **`.github/workflows/pages.yml`** + **`index.html`** — publish the site on push.

> The devcontainer runs Claude Code as the non-root `node` user. This matters: bypass mode refuses
> to run under root/sudo, and the dev container is the supported way to run autonomously.

---

## Phase 6 — Launch the agent (never-prompted)

This is the step that keeps everything off your computer.

16. In the repo, click the green **Code** button → **Codespaces** → **Create codespace**, and
    **open it in the browser** (do *not* choose "Open in VS Code Desktop"). Browser = zero local
    footprint; the agent runs entirely on GitHub's cloud VM.
17. Open the integrated terminal: **Terminal → New Terminal**, or press `` Ctrl+` ``.
18. Confirm the key is present:
    ```bash
    echo ${ANTHROPIC_API_KEY:+set}   # should print: set
    ```
19. Launch the agent (the alias already adds the bypass flag):
    ```bash
    claude
    ```
 - Select your terminal display colors
 - Choose Yes to use API key
 - Acknowledge mistakes
 - Choose Yes to recommended settings
 - Choose Yes to trust folder
 - Accept the one-time Bypass Permissions warning the first time. With the persisted `~/.claude` volume it won't reappear on later rebuilds.

> **Don't run `claude login`.** With `ANTHROPIC_API_KEY` set, Claude Code uses the capped key
> automatically. Logging in with a Pro/Max account would pull the agent *off* the capped key and
> onto subscription billing — bypassing the $20 ceiling entirely. The injected secret is all the
> auth it needs.

### Where you type prompts

Claude Code is a command-line tool, not a chat sidebar. After the launch step you drop into the
Claude Code REPL — a `>` prompt in the terminal. Type your prompt there and press Enter. In bypass
mode the agent runs every action — file edits, shell commands, network calls, git — with **no
approval prompts**. Keep typing follow-ups in the same session; `/exit` or `Ctrl+C` leaves.

For unattended, one-shot runs:
```bash
claude -p "your task here"
```

Mental model: the **browser tab** is your keyboard and screen, the **terminal panel** is your chat
window, and the `claude` process plus every command it runs executes on the **cloud VM**.

---

## Phase 7 — Publish output via GitHub Pages

20. Repo → **Settings → Pages → Build and deployment → Source** → **GitHub Actions**. The included
    `pages.yml` then publishes on every push to `main`. (Actions minutes are unlimited on public
    repos, so builds cost nothing.)
21. The site goes live at `https://<new-username>.github.io/<repo>/`. Anything the agent commits
    republishes automatically.

---

## Phase 8 — Verify the isolation (60-second check)

- [ ] Connected to the Codespace **via the browser** → nothing runs locally.
- [ ] The GitHub account has **no other repos or orgs** → nothing but this repo to reach.
- [ ] The Codespace's auto GitHub token is **repo-scoped by default**.
- [ ] The **only secret** present is the workspace-scoped key from the **separate** Anthropic account.
- [ ] **Spend caps set on both sides:** Anthropic org ($20/mo tokens) and GitHub Codespaces (compute).

**Net effect:** a brand-new identity, one public repo, a fully autonomous agent on a throwaway
cloud VM with open internet, repo-only data access, no path to your computer or real accounts, and
a hard ceiling on both bills.

---

## Never-prompted mode: what to expect

- **Bypass, not auto mode.** Bypass mode disables all permission prompts and lets Claude act
  freely. Claude Code's newer "auto mode" reduces prompts but falls back to the normal approval
  flow when a command can't be sandboxed — so for a true zero-prompt experiment, the bypass flag
  is the right choice.
- **Two residual prompt sources.** (1) A one-time acceptance the first time you launch with the
  flag in a fresh environment — a single keystroke, persisted by the config volume. (2) Direct
  edits to protected dirs (`.git/`, `.claude/`, `.vscode/`, `.husky/`) can still prompt even under
  bypass. Running `git` via Bash and writing your site files do **not** trigger this; you'd only
  hit it asking the agent to hand-edit those folders.
- **No protection, by design.** `bypassPermissions` offers no protection against prompt injection
  or unintended actions — it removes prompts, it doesn't make actions safe. The throwaway account,
  ephemeral VM, repo-only token, and capped key are what actually contain a runaway agent.

---

## Optional: lock down outbound internet

This setup leaves the agent's outbound internet **open**, which is the intent here. If you ever
want to restrict egress to a whitelist (npm, GitHub, the Anthropic API), add the firewall script
from Anthropic's reference dev container to the `.devcontainer/`. You don't need it to meet the
stated goal — the only readable things are a public repo and a capped key.

---

## Sources (official docs)

- Claude Code setup & Node.js requirement: <https://code.claude.com/docs/en/setup>
- Claude Code permission modes (bypass, root refusal, web caveat): <https://code.claude.com/docs/en/permission-modes>
- Claude Code npm package: <https://www.npmjs.com/package/@anthropic-ai/claude-code>
- Anthropic auto mode (vs bypass): <https://www.anthropic.com/engineering/claude-code-auto-mode>
- Anthropic workspaces & spend limits: <https://platform.claude.com/docs/en/manage-claude/workspaces>
- API key env var takes precedence over subscription: <https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan>
- GitHub Pages availability by plan: <https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages>
- GitHub Terms (account requirements / machine accounts): <https://docs.github.com/en/site-policy/github-terms/github-terms-of-service>
