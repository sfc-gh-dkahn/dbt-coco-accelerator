# dbt CoCo Accelerator

Turn a **Jira**, **Linear**, or **Azure DevOps** ticket into finished, tested dbt model changes on a feature branch — driven entirely by chat inside **Snowflake Cortex Code (CoCo)**.

You describe the ticket; CoCo reads it, finds the right dbt repo, plans the change with you, writes the models, validates them, and opens the PR. It uses the [dbt MCP server](https://docs.getdbt.com/docs/dbt-ai/about-mcp) plus your ticket tool's MCP server for context, and the `git`/`gh` CLIs for source control.

> [!IMPORTANT]
> **This is a personal project by Dylan Kahn, shared for community reference. It is NOT an official Snowflake, dbt Labs, Atlassian, Linear, or Microsoft product** and is not supported or endorsed by any of them. Provided as-is, with no warranty.

---

## What's in here

Four skills. You run the first one once; it leaves you with exactly the one you'll use day to day.

| Skill | Purpose |
|---|---|
| **`dbt-coco-setup`** | One-time onboarder. Auto-detects your setup, installs dbt's official skills and the MCP servers you need, authenticates GitHub, and configures your accelerator. Removes the accelerators you don't use when done. |
| **`dbt-jira-accelerator`** | Implements dbt work from a **Jira** ticket. |
| **`dbt-linear-accelerator`** | Implements dbt work from a **Linear** issue. |
| **`dbt-azure-devops-accelerator`** | Implements dbt work from an **Azure Boards** work item. |

You only keep **one** accelerator — Jira, Linear, *or* Azure DevOps — chosen during setup.

## Built on dbt's official skills

This project doesn't reinvent dbt expertise — it **orchestrates a ticket-to-PR workflow** and delegates the actual dbt craft to **dbt Labs' official, maintained skills** ([`dbt-labs/dbt-agent-skills`](https://github.com/dbt-labs/dbt-agent-skills), Apache-2.0). `dbt-coco-setup` installs the `dbt` skill bundle for you, and the accelerators defer to it during planning, implementation, and validation:

| When | Delegates to |
|---|---|
| Planning a change | `using-dbt-for-analytics-engineering`, `using-dbt-state`, `working-with-dbt-mesh` (breaking changes) |
| Writing models & tests | `using-dbt-for-analytics-engineering`, `adding-dbt-unit-test`, `building-dbt-semantic-layer` |
| Validating | `running-dbt-commands`, `troubleshooting-dbt-job-errors` |

You get dbt Labs' up-to-date expertise plus this project's ticket + git orchestration and stop-point discipline. If the dbt skills aren't installed, the accelerators still run on their own (thinner) built-in guidance.

---

## Quick start

### 1. Install the plugin into CoCo

Paste this repo's link into the CoCo chat and ask it to install the plugin, e.g.:

> Install this plugin: https://github.com/sfc-gh-dkahn/dbt-coco-accelerator

CoCo clones the repo and registers all four skills.

### 2. Run setup (one-time, ~5 minutes)

> Run the **dbt-coco-setup** skill.

It's built to be fast: it **auto-detects almost everything** (your OS, dbt project/repo, default branch, GitHub login, prerequisites) and only asks what it genuinely can't figure out. For a local dbt user that's about **two taps**:

1. **One question:** which ticket tool (**Jira**, **Linear**, or **Azure DevOps**) and how you run dbt — **Local** (dbt Core, Fusion, or dbt Projects on Snowflake) or **dbt Cloud**. *(dbt Cloud users paste their MCP URL and sign in via browser — no token, unless OAuth isn't enabled on their account. Azure DevOps users also give their org and project name, and pick a sign-in method — see the note below.)*
2. **One review-and-confirm:** setup shows everything it detected and exactly what it'll change (install dbt Labs' official `dbt` skills, add `dbt` + your ticket server to `mcp.json` — existing servers untouched, install any missing prerequisites). You hit **Proceed**. Anything destructive — replacing or removing a server you already had — is asked separately, never bundled into that one gate.

It then removes the accelerators you won't use automatically and invites you to try it immediately. (If the new tools don't show up on first try, restart Cortex Code once — MCP tools load at session start.)

### 3. Use your accelerator

Just describe the ticket in chat — the skill triggers on its own:

> Pick up ticket **DE-1234** (Jira) &nbsp;/&nbsp; Work on issue **ENG-123** (Linear) &nbsp;/&nbsp; Pick up work item **1234** (Azure DevOps)

Or invoke it explicitly with a slash command:

> `/dbt-jira-accelerator` pick up ticket DE-1234
>
> `/dbt-linear-accelerator` work on issue ENG-123
>
> `/dbt-azure-devops-accelerator` pick up work item 1234

The accelerator reads the ticket, confirms understanding, plans with you, implements, validates, and opens a PR.

---

## Prerequisites

Setup checks these for you and offers to install what's missing. Listed here for transparency:

- **Snowflake Cortex Code** (this is where the skills run).
- **`git`** and the **GitHub CLI (`gh`)**, authenticated to your GitHub account.
- **A dbt project you already work in** (a repo with `dbt_project.yml` and a working `profiles.yml`). These skills implement changes in *your* dbt project — they don't create one from scratch.
- **For dbt Core (local MCP):** [`uv`/`uvx`](https://docs.astral.sh/uv/) so CoCo can run `uvx dbt-mcp`.
- **For dbt Cloud (remote MCP):** a dbt platform account. Setup prefers **OAuth** (just your MCP Endpoint URL + a browser sign-in); a **service token** is only needed as a fallback where OAuth isn't available.
- **A Jira Cloud site**, a **Linear workspace**, *or* an **Azure DevOps Services organization**, depending on which accelerator you choose.
- **For Azure DevOps:** **Node 20+** (Microsoft's MCP server runs via `npx`), plus the **Azure CLI** — the recommended path reuses your `az login` and stores no token. A **Personal Access Token** works as an advanced fallback, set as an environment variable in your own shell rather than written into any config file. Azure DevOps **Server (on-premises) is not supported** by Microsoft's MCP server — Services only.

---

## The MCP servers this installs

Setup only adds the servers you need — dbt plus your one ticket tool.

| Server | Transport | Auth |
|---|---|---|
| **dbt (Core)** | local `stdio` via `uvx dbt-mcp` | local dbt profile / env vars — no browser |
| **dbt (Cloud)** | remote `http` | **browser OAuth** (just the MCP URL) — service token only as fallback |
| **Atlassian / Jira** | remote `http` (`https://mcp.atlassian.com/v1/mcp`) | **browser OAuth** on first connect |
| **Linear** | remote `http` (`https://mcp.linear.app/mcp`) | **browser OAuth** on first connect |
| **Azure DevOps** | local `stdio` via `npx -y @azure-devops/mcp@2.10.0 <org>` | **Azure CLI** (`az login`) — no token stored; PAT via your own env var as a fallback |

GitHub is **not** an MCP server here — the skills use the `git` and `gh` command-line tools directly. That's true for the Azure DevOps flow too: the **work item** lives in Azure Boards, but branches and PRs go to **GitHub**. Azure Repos isn't used.

The Azure DevOps server is **version-pinned** (`@2.10.0`). Microsoft renamed its entire tool surface once already, so an unpinned `npx` install can change behaviour underneath you. Pinning `@2.8.1` instead restores the older flat tool names if you need them.

### A note on Azure DevOps sign-in

Jira and Linear authorize with one browser click. Azure DevOps doesn't, and it's worth knowing why before you pick it.

Microsoft ships two servers. The **hosted (remote)** one authenticates through Microsoft Entra and needs *dynamic OAuth client registration* — something most desktop clients can't do, so Microsoft documents Claude Desktop, Claude Code, Cursor, and Codex as needing either the local server or a hand-built Entra app registration. It also rejects standalone Microsoft-account organizations. Setup therefore defaults to the **local** server with Azure CLI credentials, which just works if you've run `az login`. You can still choose hosted; setup will offer to fall back to local if it won't authorize, and will ask before changing anything it didn't create.

If you use a **PAT** instead, setup will not ask you to paste it. You export `PERSONAL_ACCESS_TOKEN` in your own shell profile and the server reads it from the environment — following Microsoft's own instruction: *"Do not commit tokens to an MCP configuration file. Set them outside the file or use a secrets manager."*

One more thing to know: the current tools are dispatchers (`wit_work_item` with an `action` parameter) rather than the older flat names (`wit_get_work_item`). The accelerator reads whichever generation is actually in your session, so both work — but if you see tool-name errors, that rename is the first thing to check.

---

## Known unknowns (Azure DevOps)

**The Azure DevOps accelerator has not yet been run end to end against a live Azure DevOps organization.** Jira and Linear have been; this one is built entirely from Microsoft's public documentation. That's a real difference and worth stating plainly rather than burying.

Everything documented above is sourced from Microsoft's docs. These specific points could not be settled from documentation alone, so the skill carries a defined fallback for each instead of assuming:

| Assumption | If it's wrong |
|---|---|
| `cortex mcp add` forwards `-y` and the `-d` domain list through to `npx` | Setup reads the stored `args` array back and tells you to correct `mcp.json` by hand. **This is the most likely install snag.** |
| CoCo passes its environment to the MCP child process, so the PAT path can read `PERSONAL_ACCESS_TOKEN` | The accelerator reports the auth error and points you to the Azure CLI path, which needs no env var. |
| The pinned server exposes the dispatcher tool names as documented | The skill enumerates the tools actually present, tries the legacy names, then stops and reports rather than guessing. |
| The hosted/remote server can complete sign-in inside CoCo | Setup says so and offers the local path — asking first before removing anything. |
| Your project uses a documented process template (Agile/Scrum/Basic) | State names are read from the work item type at run time; if that fails, the skill asks you. |

If you run this against a real organization, corrections are welcome — that's the fastest way to turn these into facts.

---

## Privacy & secrets

- Your workflow answers and any tokens are written **only to local files** on your machine (`mcp.json` and the skill file). Nothing is sent anywhere by these skills.
- **dbt Cloud (token fallback only):** if OAuth isn't available and you use a service token, it's stored in plaintext in `mcp.json`. Treat that file as a secret and **never commit it**. (The default OAuth path stores no token.)
- **Azure DevOps:** the recommended Azure CLI path stores **no token at all**. If you use a PAT, setup never asks you to paste it and never writes it to `mcp.json` — you export it in your own shell and the server reads it from the environment, per Microsoft's guidance: *"Do not commit tokens to an MCP configuration file."*
- **Third-party MCP packages are version-pinned** so an upstream change can't silently alter behaviour on your machine.
- **Prompt injection:** ticket text is untrusted input. The accelerators treat descriptions, comments, and linked pages as data to analyze, never as instructions — a published attack against ADO-connected agents uses hidden PR/work-item comments to redirect them. Stop-points exist partly for this reason; don't remove them.
- This repository contains **no credentials**. Don't add any.

---

## Updating

Re-install the plugin from CoCo to pull the latest skills. Re-running `dbt-coco-setup` is safe — it detects an existing setup and offers to repair or update your configuration instead of duplicating anything.

---

## Disclaimer

This is a **personal project shared for community reference**. It is **not an official Snowflake, dbt Labs, Atlassian, Linear, or Microsoft product** and is not supported or endorsed by any of them. These skills install software, edit local config, and run `git`/`dbt` commands — review each `SKILL.md` before installing, and use at your own discretion.
