---
name: dbt-coco-setup
description: "One-time setup for the dbt CoCo Accelerator. Auto-detects what it can, installs dbt Labs' official skills and the MCP servers you need (dbt + Jira, Linear, or Azure DevOps), authenticates GitHub, and configures your chosen accelerator — with a single review-and-confirm gate. Use when: first-time setup, install dbt accelerator, configure dbt jira, configure dbt linear, configure dbt azure devops, set up dbt ticket accelerator, onboard dbt accelerator. Triggers: dbt coco setup, setup dbt accelerator, install dbt accelerator, configure accelerator."
---

# dbt CoCo Accelerator — Setup

One-time setup that makes the `dbt-jira-accelerator`, `dbt-linear-accelerator`, or `dbt-azure-devops-accelerator` skill work end to end.

## Design intent (how the agent must run this)

Installation is the biggest barrier to adoption, so this skill is engineered for completion, not thoroughness theater:

- **Auto-detect and default aggressively.** Ask the user *only* what you genuinely cannot determine yourself. Everything detectable (OS, repo/project path, default branch, `gh` auth, prerequisites) you detect silently.
- **Front-load the tiny bit of required input,** then do the detective work, then present **one** review-and-confirm gate before making any changes.
- **Batch all changes behind that single consent.** Do not gate each action separately.
- **Determinism rule:** where a step gives an **"Ask exactly this"** block, present that `ask_user_question` verbatim; substitute only `<PLACEHOLDERS>`.
- **Required actions are "Proceed" / "Cancel setup" — never a misleading "No."** If the user cancels, explain what won't work.
- **Never silently mutate `mcp.json`, secrets, or skills.** Show exactly what will change at the review gate first.
- **End on capability, not a chore:** invite them to try the accelerator immediately. On current builds `cortex mcp add` makes new tools available automatically; confirm with `cortex mcp start` (and `cortex mcp reconnect` to retry a stuck server) — run these **yourself**, don't ask the user to reload/restart. A reload is a last resort only on older (pre-1.0.65) builds without in-session MCP refresh.

Keep the tone light and encouraging. This should feel like ~5 minutes, mostly you doing the work.

---

## Step 0: Re-run guard (only matters on a second run)

Check for the marker file (`~/.snowflake/cortex/.dbt-coco-accelerator.json`; Windows `%USERPROFILE%\.snowflake\cortex\.dbt-coco-accelerator.json`).

- **If absent:** fresh install — go to the Intro.
- **If present, Ask exactly this** (fill `<PLACEHOLDERS>` from the file):
  - header: `Existing setup`
  - question: `This machine already has the dbt CoCo Accelerator set up (<TICKET_TOOL>, dbt <FLAVOR>, configured <DATE>). What would you like to do?`
  - options:
    - `Repair / re-verify` — Re-check everything and fix what's missing. No duplicates created.
    - `Update workflow context only` — Re-detect and rewrite the context block in your accelerator.
    - `Reconfigure from scratch` — Run setup again; existing `mcp.json` servers are preserved.
    - `Cancel` — Make no changes.

---

## Intro (say this first — sets expectations + motivation)

Tell the user, warmly and briefly:

> **One-time install, ~5 min. It will be worth it.** — Dylan
>
> I'll auto-detect most of it and only ask what I can't figure out.

Then continue.

---

## Phase 1 — What only you can tell me

**Goal:** Collect the few things that can't be detected. Front-load all human input here.

**Ask exactly this** (both questions in one `ask_user_question` call):

- Question 1:
  - header: `Ticket tool`
  - question: `Which ticket tool do you use for your dbt work?`
  - options:
    - `Jira` — Atlassian Jira Cloud.
    - `Linear` — Linear issues.
    - `Azure DevOps` — Azure Boards work items (Azure DevOps **Services**; on-premises Server isn't supported by Microsoft's MCP).
- Question 2:
  - header: `dbt setup`
  - question: `How do you run dbt?`
  - options:
    - `Local dbt project` — You have (or can clone) a git-backed dbt project — dbt Core, the dbt Fusion engine, or **dbt Projects on Snowflake** (even if you authored it in Snowsight; I'll clone the repo and set up local dbt). Uses the local dbt MCP server (uvx dbt-mcp).
    - `dbt Cloud / platform (remote)` — Hosted dbt with no local project. Uses the remote dbt MCP server over HTTP. **Prefers OAuth** (just an MCP URL + browser sign-in, like Jira/Linear); falls back to a service token only if OAuth isn't available on the account.

**If — and only if — they chose dbt Cloud:** prefer **OAuth** (no token, no plaintext secret). **Ask exactly this** (one field):

- header: `dbt MCP URL`
- question: `Paste your dbt platform MCP Endpoint URL — find it under Account settings → Access URLs → MCP Endpoint URL (e.g. https://abc123.us1.dbt.com/api/ai/v1/mcp). You'll sign in via your browser on first connect — no token needed.`
- type: text
- defaultValue: `https://<your-prefix>.dbt.com/api/ai/v1/mcp`

**Token fallback — only if the user says OAuth isn't enabled on their dbt account** (OAuth is Beta and requires a Starter/Enterprise+ plan). **Ask exactly this** (one call, four fields):

- Question 1: header `dbt host`, question `Your dbt platform host (in your browser's address bar when logged into dbt), e.g. cloud.getdbt.com or abc123.us1.dbt.com.`, type text, defaultValue `cloud.getdbt.com`
- Question 2: header `dbt service token`, question `A dbt service token: Account settings → API tokens → Service tokens (Metadata + Semantic Layer + Developer perms). NOTE: stored in plaintext in your local mcp.json — never commit that file.`, type text, defaultValue `` (empty)
- Question 3: header `Prod env ID`, question `Production environment ID — the number in .../environments/<ID> for your prod/deploy environment.`, type text, defaultValue `` (empty)
- Question 4: header `Dev env ID`, question `Development environment ID — the number in .../environments/<ID> for your dev environment.`, type text, defaultValue `` (empty)

**If — and only if — they chose Azure DevOps:** you need the organization name, which can't be detected. **Ask exactly this** (one call, two fields):

- Question 1: header `ADO org`, question `Your Azure DevOps organization name — the segment after dev.azure.com in your URL (e.g. for https://dev.azure.com/contoso/MyProject it's just: contoso).`, type text, defaultValue `` (empty)
- Question 2: header `ADO project`, question `The Azure DevOps project your dbt work items live in (e.g. Data Platform). Work item IDs are bare numbers, so the accelerator needs the project name on every lookup.`, type text, defaultValue `` (empty)

**Then pick the auth path. Azure DevOps is the one ticket tool without a clean browser-OAuth story** — Microsoft's *remote* server authenticates through Microsoft Entra and requires dynamic OAuth client registration that most desktop clients (and likely CoCo) can't do; it also refuses standalone Microsoft-account orgs. The **local** server is the reliable default. **Ask exactly this:**

- header: `ADO sign-in`
- question: `How should I connect to Azure DevOps? The local server is the safe default — Microsoft's hosted server needs an Entra app registration that most desktop clients can't do yet.`
- options:
  - `Azure CLI (recommended)` — Runs Microsoft's server locally and reuses your `az login` session. No token to paste or store. Needs the Azure CLI and Node 20+; I'll install what's missing.
  - `Personal Access Token` — Runs the server locally with an ADO PAT. Works without the Azure CLI, but the token sits in plaintext in your local `mcp.json`.
  - `Hosted (remote) server` — Microsoft's hosted endpoint. Fewest moving parts *if* your org is Entra-backed and CoCo can complete the sign-in — but it may fail on the OAuth step, and we'd fall back to local.

If they pick **Personal Access Token**, ask for it (header `ADO PAT`, type text, empty default) and note it stays local and must never be committed. If they pick **Hosted**, warn once that you'll fall back to Azure CLI local if the connection won't authorize — then carry on without further questions.

That is the last of the required typing. Everything else you detect or default.

---

## Phase 2 — Detect silently (no prompts)

Work these out yourself and hold the results for the review gate. Do **not** ask the user for anything you can detect:

1. **OS** — `uname -s` (macOS/Linux) or Windows. Drives paths/installers:
   | | macOS | Linux | Windows |
   |---|---|---|---|
   | config | `~/.snowflake/cortex/mcp.json` | `~/.snowflake/cortex/mcp.json` | `%USERPROFILE%\.snowflake\cortex\mcp.json` |
   | install `uv` | `brew install uv` | `curl -LsSf https://astral.sh/uv/install.sh \| sh` | `winget install astral-sh.uv` |
   | install `gh` | `brew install gh` | https://github.com/cli/cli#installation | `winget install GitHub.cli` |
2. **dbt project + repo path** (local flavor) — `find ~ . -maxdepth 4 -name dbt_project.yml 2>/dev/null`. This folder is also the **git repo** (the accelerator branches/commits here). If several are found, keep them for the review gate and let the user pick there. On Windows, if `find` is unavailable, this is one of the few things you may ask. **If none is found**, the user may author in **Snowsight (dbt Projects on Snowflake)** — still git-backed, just not cloned locally; ask for the repo URL so you can `git clone` it in Phase 4 (the project often lives in a subfolder).
3. **Default branch** — `git -C <repo> symbolic-ref --short refs/remotes/origin/HEAD` (fallback `main`).
4. **`gh` presence + auth** — `gh --version`, `gh auth status`.
5. **`uv`/`uvx`** (local flavor only) and **`git`** — `command -v`.
6. **Azure DevOps prerequisites** (ADO flow only) — `node --version` (Microsoft's server needs **Node 20+**) and `npx --version`; if they chose Azure CLI auth, also `az --version` and `az account show` to see whether they're already signed in. Hold anything missing for the review gate (install `node` via `brew install node` / `winget install OpenJS.NodeJS`; `az` via `brew install azure-cli` / `winget install Microsoft.AzureCLI`).
7. **Existing MCP servers** — run `cortex mcp list` (and/or read `mcp.json`) to see what's already configured, and check whether a working server already covers what you'd add: a **Jira/Atlassian** server (any name — `atlassian`, `atlassian-remote`, a Snowflake-hosted one) for the Jira flow, a **Linear** server for the Linear flow, an **Azure DevOps** server (`ado`, `azure-devops`) for the ADO flow, or an existing **dbt** server. If one exists and is connected, plan to **reuse it — don't add a duplicate**. Never print secrets back to the user.

**Defaults to assume** (no need to ask; the user can change them at the review gate):
- Branch naming: `feature/<KEY>-<short-description>`
- Commit convention: conventional commits `feat(<area>): <description> [<KEY>]`
- PR method: `gh` CLI (auto-open PR)
- Dev/prod target: **not asked** — the accelerator checks `dbt debug` live at validation time.
- Jira/Linear site, project/team keys, workflow states: **not asked** — the accelerator discovers these live per-ticket (ticket fetch + transitions/state APIs).
- Azure DevOps process template and state names (Agile `Active`, Scrum `Committed`, Basic `Doing`, or custom): **not asked** — the accelerator reads the allowed `System.State` values from the work item type at run time. Org and project *are* asked, because a bare work item ID doesn't identify either.

---

## Phase 3 — Review & confirm (the single gate)

**Goal:** Show the user how much is already done and get one consent before any change. This is the completion moment — lead with what you've already handled.

Present a compact summary, e.g.:

```
Here's what I've set up for you automatically:
  ✓ OS: macOS
  ✓ dbt project / repo: <path>
  ✓ Default branch: <branch>
  ✓ GitHub CLI: installed, logged in as <user>
  ✓ Prerequisites: uv ✓, git ✓
Defaults I'll use (you can change any):
  • Branch names: feature/<KEY>-<slug>   • Commits: conventional   • PRs: gh CLI
  • Ticket-tool specifics (states, workflows): discovered automatically per ticket
When you proceed, I will:
  1. Install dbt Labs' official dbt skills as a syncable plugin (dbt-labs/dbt-agent-skills)
  2. Add only the MCP servers you don't already have (existing servers untouched; a Jira/Linear/dbt server you already have is reused, not duplicated):
       + dbt        (<local uvx | remote http>)   ← only if not already present
       + <atlassian | linear | ado>               ← only if not already present
  3. [Install any missing prerequisites: <list>]  ← only if something's missing
  4. Save your setup into the <accelerator> skill
```

If anything required is missing (e.g. `uv`, `gh`, or `gh` not authed), name it here and note it's part of "Proceed."

**Ask exactly this:**
- header: `Ready to set up`
- question: `I've auto-detected everything above and I'm ready to apply it. Proceed?`
- options:
  - `Proceed — set it up` — Applies everything listed above in one go.
  - `Adjust something first` — Lets you change the repo path, branch, or any default before applying.
  - `Cancel setup` — Makes no changes. (The accelerator won't work until setup completes.)

**If "Adjust something first":** ask (with text fields pre-filled with the detected/default values) only for what they want to change, then re-show the summary and this same Proceed prompt. Keep the default path frictionless — only reveal override fields on request (progressive disclosure).

---

## Phase 4 — Apply (batch; no further gates)

The user consented at Phase 3. Execute in order, reporting each result briefly:

1. **Missing prerequisites & GitHub sign-in.** Install any missing prereqs (`uv`, `git`, `gh` itself) using the OS command from Phase 2 — those you *can* run for them.
   **GitHub auth is the one thing you cannot do for them.** Only if `gh auth status` (Phase 2) showed *not* authenticated: `gh auth login` needs an interactive terminal + browser and will hang if you try to run it yourself (no TTY — don't attempt it, and don't tell them to run it "in the chat" or with a `!` prefix; the chat isn't a terminal for an interactive login). Hand it off — **Ask exactly this:**
   - header: `GitHub sign-in`
   - question: `One quick step needs your own terminal. Open a terminal OUTSIDE this chat — your macOS Terminal app, or a separate terminal tab/window — and run:  gh auth login  then choose GitHub.com → HTTPS → "Login with a web browser" and finish in the browser. It's interactive, so I can't run it for you. Tell me when you're signed in.`
   - options:
     - `Done — I'm signed in` — I'll verify and keep going.
     - `Cancel setup` — Stops here; without GitHub auth the accelerator can't push branches or open PRs.
   After they pick "Done," confirm with `gh auth status`. If it still fails, show the command again and re-ask — never proceed unauthenticated.
2. **dbt skills — install as a *syncable plugin* (preferred), not loose copies.** Install `dbt-labs/dbt-agent-skills` via CoCo's GitHub plugin installer (`https://github.com/dbt-labs/dbt-agent-skills`; the `skills/dbt` bundle if the installer supports a subpath, otherwise the whole repo — the extra `dbt-migration`/`dbt-extras` skills are harmless). Installing it as a **plugin** registers it in `registry.json` with a **Sync** button — that's what lets the user pull dbt's upstream updates later. Verify at least `using-dbt-for-analytics-engineering` and `running-dbt-commands` registered.
   - **Fallbacks if the plugin install isn't available:** `npx skills add dbt-labs/dbt-agent-skills/skills/dbt --global`; only as a **last resort**, clone and copy `skills/dbt/skills/<name>/` into the CoCo skills dir — and if you copy, **tell the user those are frozen snapshots that won't auto-update** (re-running setup can refresh them).
   - **On a re-run / Repair:** if you find loosely-copied dbt skills from a prior install (present in the user skills dir with no registry entry), offer to convert them to the synced plugin and de-duplicate.
3. **Add the MCP servers with `cortex mcp add`** — use this CLI, not a hand-edit of `mcp.json`. It merges into `~/.snowflake/cortex/mcp.json` **without touching existing servers** *and* registers each server with the running MCP manager, which is what lets `cortex mcp reconnect` connect them live in Phase 6 (a hand-edited JSON entry the manager never loaded can't be reconnected in-session). Add only the dbt server + the chosen ticket server, and **only the ones not already present** — if Phase 2 found a working Atlassian/Jira, Linear, or dbt server, reuse it and skip that `add`. Run these yourself:
   - **dbt Local:**
     ```bash
     cortex mcp add dbt uvx dbt-mcp -e DBT_PROJECT_DIR="<REPO_PATH>" -e DBT_PATH=dbt
     ```
     Handle these first only if they apply (most local users need none):
     - **No local clone yet** (Snowsight-authored dbt Projects on Snowflake): `git clone` their repo; `DBT_PROJECT_DIR` is the project folder (often a subfolder).
     - **No local `dbt` binary** (`dbt-mcp` crashes if `DBT_PATH` isn't found — check `command -v dbt`/`dbtf`): install it (`uv tool install dbt-core --with dbt-snowflake`) and pass its real path, or add `-e DISABLE_DBT_CLI=true` for a compile-only setup (validation then relies on a Snowflake run).
     - **Placeholder `profiles.yml`** (built to run in Snowflake): create `~/.dbt/profiles.yml` from their existing `snow` connection — prefer key-pair or `authenticator: externalbrowser` over a plaintext password, `chmod 600`, keep it **out of the git repo**; confirm with `dbt debug`.
   - **dbt Cloud — OAuth (preferred):** URL only, http transport; CoCo does the browser OAuth on first connect, like Jira/Linear.
     ```bash
     cortex mcp add dbt "<MCP_ENDPOINT_URL>" --type http
     ```
     (Ref: https://docs.getdbt.com/docs/platform/manage-access/connect-apps-oauth)
   - **dbt Cloud — token (fallback, only if OAuth unavailable):**
     ```bash
     cortex mcp add dbt "https://<DBT_HOST>/api/ai/v1/mcp/" --type http \
       -H "Authorization: token <TOKEN>" \
       -H "x-dbt-prod-environment-id: <PROD_ENV_ID>" \
       -H "x-dbt-dev-environment-id: <DEV_ENV_ID>"
     ```
     (Token lands plaintext in the local `mcp.json` — never commit it. Re-check https://docs.getdbt.com/docs/dbt-ai/about-mcp if the endpoint looks stale.)
   - **Jira:** `cortex mcp add atlassian https://mcp.atlassian.com/v1/mcp --type http`
   - **Linear:** `cortex mcp add linear https://mcp.linear.app/mcp --type http`
   - Jira/Linear (and dbt Cloud OAuth) authenticate via **browser on first connect** — no token here.
   - **Azure DevOps — local + Azure CLI (recommended):** Microsoft's server is an npx package that takes the org as its first positional argument. Load only the tool groups the accelerator needs (`-d`) — the full surface spans repos, pipelines, test plans and wiki, and Microsoft themselves note models do worse with a bloated tool list:
     ```bash
     cortex mcp add ado npx -- -y @azure-devops/mcp <ADO_ORG> --authentication azcli -d core work work-items search
     ```
     Then confirm the CLI session exists — `az account show`; if it doesn't, the user must run `az login` in **their own terminal** (interactive browser flow, same constraint as `gh auth login` — hand it off, don't try to run it yourself).
   - **Azure DevOps — local + PAT (fallback):** same command, `--authentication pat`, with the token base64-encoded as `email:pat` in `PERSONAL_ACCESS_TOKEN`:
     ```bash
     cortex mcp add ado npx -- -y @azure-devops/mcp <ADO_ORG> --authentication pat -d core work work-items search \
       -e PERSONAL_ACCESS_TOKEN="$(printf '%s' '<EMAIL>:<PAT>' | base64)"
     ```
     (Plaintext in the local `mcp.json` — warn, never commit.)
   - **Azure DevOps — hosted/remote (only if they chose it):** `cortex mcp add ado https://mcp.dev.azure.com/<ADO_ORG> --type http -H "X-MCP-Toolsets: wit,search"` — then **verify it actually authorizes**. Microsoft's remote server needs Entra dynamic OAuth client registration, which most desktop clients can't do, and it rejects standalone Microsoft-account orgs. If it won't connect, say so plainly, remove it (`cortex mcp remove ado`), and fall back to the local + Azure CLI command above.
   - **If `cortex mcp add` mangles the npx flags** (its own parser may swallow `-y`/`-d`): drop the `--` separator or, as a last resort, add the server and then correct the `args` array in `~/.snowflake/cortex/mcp.json` by hand — then run `cortex mcp start` so the manager loads it. Verify either way with `cortex mcp list`.
   - Then run `cortex mcp list` to confirm the new servers sit alongside the existing ones (none clobbered).
4. **Context block** — write the user's setup into the chosen accelerator's `SKILL.md`, replacing anything between the idempotent markers (insert once if absent):
   ```
   <!-- USER_WORKFLOW_CONTEXT:START -->
   ## Workflow Context (auto-generated by dbt-coco-setup — safe to re-generate; do not hand-edit)

   - Ticket tool: <Jira|Linear|Azure DevOps>
   - ADO organization / project: <org> / <project>   (Azure DevOps only)
   - ADO auth: <azcli|pat|remote>                    (Azure DevOps only)
   - dbt flavor: <Local|dbt Cloud>
   - dbt project / repo path: <path>   (Local only)
   - Default branch: <branch>
   - Branch naming: <pattern>
   - Commit convention: <pattern>
   - PR method: <gh CLI|push-only>
   - Notes: dev/prod target checked live via `dbt debug`; ticket site/keys/states discovered per-ticket
   <!-- USER_WORKFLOW_CONTEXT:END -->
   ```

Report a concise "Done: installed dbt skills, added dbt + <tool> via `cortex mcp add`, saved your setup."

---

## Phase 5 — Trim (automatic — do NOT ask)

The user already told you which tool they use, so the other accelerator is obviously unnecessary. **Remove it automatically — do not prompt.**

- Delete the **other two** accelerator folders from the installed plugin dir (e.g. if they chose Jira, remove `dbt-linear-accelerator/` and `dbt-azure-devops-accelerator/`).
- **Keep** `dbt-coco-setup` installed so they can re-run it later to repair or reconfigure (it won't trigger by accident).
- **Never remove dbt's official skills.**
- **Announce** what happened (transparency, not a question), e.g.: *"Removed the unused Linear and Azure DevOps accelerators. You now have `dbt-jira-accelerator` + dbt's official skills. (Kept the setup skill in case you want to reconfigure later.)"*
- If `skills.json` listed the removed skill, it clears from the picker on the next session — harmless if it lingers until then; no action needed.

---

## Phase 6 — Activate & try it now (end on a win)

1. Write the marker file (no prompt — local only, no secrets):
   ```json
   { "configured_at": "<ISO>", "ticket_tool": "<jira|linear|azure-devops>", "dbt_flavor": "<local|cloud>",
     "mcp_servers_added": ["dbt", "<atlassian|linear|ado>"], "dbt_skills_installed": true,
     "accelerator_skill": "<dbt-jira-accelerator|dbt-linear-accelerator|dbt-azure-devops-accelerator>",
     "ado": { "org": "<org>", "project": "<project>", "auth": "<azcli|pat|remote>" },  // ADO only
     "version": "0.1.0" }
   ```
2. **Activate the servers yourself — don't make the user reload.** On current CoCo builds (v1.0.65+ added in-process MCP refresh) `cortex mcp add` already makes a new server's tools available to the agent automatically. Connect and verify them in this session by running (yourself — normal shell commands):
   ```bash
   cortex mcp start      # connects the new server(s) and exposes their tools (the documented path)
   cortex mcp list       # confirm they sit alongside existing servers
   ```
   If a server shows disconnected/failed, run `cortex mcp reconnect` to retry it. A one-time reload (Desktop: Command Palette → "Developer: Reload Window"; CLI: restart) is a **last resort only on older (pre-1.0.65) builds** without in-session refresh — never lead with it.
3. **Show a short completion summary** in this exact style (checkmarks, tight — substitute `<PLACEHOLDERS>`; omit the prereq line if nothing was installed):

   ```
   ✅ All set — here's what's installed:
   • ✅ dbt Labs' official dbt skills
   • ✅ dbt + <Jira|Linear|Azure DevOps> connected in your MCP config
   • ✅ Your setup saved to <dbt-jira-accelerator|dbt-linear-accelerator|dbt-azure-devops-accelerator>

   Next steps:
   1️⃣ Start your first ticket by invoking the accelerator directly:
        /dbt-jira-accelerator pick up <DE-123>
        (or type  $dbt-jira-accelerator  to pick it from the list)

   (First use opens a browser once to authorize <Jira|Linear>. — omit for Azure DevOps
    with Azure CLI or PAT auth; that's already authorized.)
   ```

   **Invoke it explicitly** — say this once: casual phrasing like "pickup DE-123" may not reliably trigger the skill, and a half-trigger skips the guided stop-points. Leading with `/dbt-jira-accelerator` (or `$dbt-jira-accelerator`) guarantees the full workflow runs. Keep the summary this short — no extra sections.

---

## Notes

- Writes only to **local files** (`mcp.json`, the accelerator `SKILL.md`, the marker). Sends nothing anywhere.
- Add MCP servers with **`cortex mcp add`** (additive — never clobbers existing servers); on current builds their tools become available automatically. Confirm with **`cortex mcp start`** (and **`cortex mcp reconnect`** to retry a stuck one) — run these yourself; don't hand-edit `mcp.json` or ask the user to reload. Any token stays only in the local `mcp.json` (plaintext — warn, never commit).
- Ask only what you can't detect; present one consent gate; end by inviting them to try it.
- Present every **"Ask exactly this"** block verbatim; substitute only `<PLACEHOLDERS>`.
