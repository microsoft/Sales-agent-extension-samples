# Sales Agent Preview — Extensibility SDLC Template

A **parameterized, non-production** Microsoft 365 declarative-agent template that mirrors the base Sales agent so teams can **develop and validate extensibility** (custom tools, actions, and knowledge) through a proper software development lifecycle — **Dev → UAT → SIT** — *before* the same extensions are applied to the **production** Sales agent.

> ⚠️ **This is a TEST / NON-PRODUCTION agent.** Never publish it as your production Sales agent and never point it at real customer/production data. The agent renders a non-prod disclaimer at the start of every conversation.

---

## Contents

- [Walkthrough video](#walkthrough-video)
- [Why this template exists](#why-this-template-exists)
- [Prerequisites](#prerequisites)
- [Install the Microsoft 365 Agents Toolkit (ATK)](#install-the-microsoft-365-agents-toolkit-atk)
  - [Option A — VS Code extension (recommended)](#option-a--vs-code-extension-recommended)
  - [Option B — ATK CLI (for pipelines / headless)](#option-b--atk-cli-for-pipelines--headless)
- [Supported extensibility — test ONLY what production accepts](#supported-extensibility--test-only-what-production-accepts)
- [What's in the package](#whats-in-the-package)
  - [The templatized tokens](#the-templatized-tokens)
- [Update an existing agent instead of creating a new one](#update-an-existing-agent-instead-of-creating-a-new-one)
  - [Step 1 — Get the latest env file](#step-1--get-the-latest-env-file)
  - [Step 2 — Recover the IDs when the env file is blank](#step-2--recover-the-ids-when-the-env-file-is-blank)
  - [Step 3 — Sign in to the same tenant](#step-3--sign-in-to-the-same-tenant)
  - [Step 4 — Make sure you're allowed to update the app](#step-4--make-sure-youre-allowed-to-update-the-app)
  - [Step 5 — Fill in the env file and commit it](#step-5--fill-in-the-env-file-and-commit-it)
  - [Step 6 — Bump the version](#step-6--bump-the-version)
  - [Step 7 — Provision and publish](#step-7--provision-and-publish)
  - [Step 8 — Verify you updated rather than duplicated](#step-8--verify-you-updated-rather-than-duplicated)
  - [Pre-flight checklist](#pre-flight-checklist)
  - [If you're the first person to deploy a stage](#if-youre-the-first-person-to-deploy-a-stage)
- [Quick start](#quick-start)
  - [1. Manage your environments with `env/.env.*`](#1-manage-your-environments-with-envenv)
    - [Enabling the Dynamics/tool variables](#enabling-the-dynamicstool-variables)
  - [2. Add your external tools with ATK](#2-add-your-external-tools-with-atk)
  - [3. Provision / build for a specific environment](#3-provision--build-for-a-specific-environment)
  - [4. Test the extension — sideload or publish](#4-test-the-extension--sideload-or-publish)
  - [4b. Ship a new version of the same agent](#4b-ship-a-new-version-of-the-same-agent)
  - [5. Promote](#5-promote)
- [Evaluating the agent (optional)](#evaluating-the-agent-optional)
- [Guardrails](#guardrails)
- [Troubleshooting](#troubleshooting)
  - [Duplicate agents in the tenant after every deploy](#duplicate-agents-in-the-tenant-after-every-deploy)
  - [`TeamsAppNotExists` when publishing](#teamsappnotexists--app-with-id--does-not-exist-in-developer-portal-when-publishing)
  - [Warning: "Short name contains Beta environment keywords"](#warning-short-name-contains-beta-environment-keywords-stagstagingpreview)
  - [`manifest.MissingEnvironmentVariablesError` during packaging](#manifestmissingenvironmentvariableserror-during-packaging)
  - [Wrong tenant / agent not showing up](#wrong-tenant--agent-not-showing-up)
- [References](#references)

---

## Walkthrough video

<video src="https://github.com/microsoft/Sales-agent-extension-samples/raw/users/brijshah/Sales-Agent-Extensibility-Template/SalesAgentExtensibilitySDLC/docs/Sales-Agent-SDLC-Guide.mp4" controls width="960" height="540"></video>

> If the player doesn't load, download or open [docs/Sales Agent SDLC Guide.mp4](docs/Sales%20Agent%20SDLC%20Guide.mp4) directly.

---

## Why this template exists

The production Sales agent is **tenant-scoped** with a **single slot** and a **single overlay**, and overlays have **no precedence/priority** — so you cannot safely stage or "canary" changes on the live agent (users in overlapping groups get merged/undefined results). This template gives you an **isolated agent identity per environment** so each stage of your lifecycle is a separate, clearly-labelled, non-prod agent you can test in isolation.

| Isolation approach | What it gives you |
| --- | --- |
| **Separate agent identity per env** (this template, via `APP_NAME_SUFFIX` + env files) | An isolated **Dev / UAT / SIT** agent that never touches the production Sales agent. |
| **Separate tenant** (recommended for parallel stages) | Truly parallel stages when one tenant isn't enough. |
| **Backend env routing** (your `TOOL_BASE_URL` / `TOOL_CLIENT_ID`) | Your tools call the matching non-prod Dynamics/Power Automate/MCP endpoint per environment. |

---

## Prerequisites

- [Node.js](https://nodejs.org/) 22
- A [Microsoft 365 account for development](https://docs.microsoft.com/microsoftteams/platform/toolkit/accounts) with a [Microsoft 365 Copilot license](https://learn.microsoft.com/microsoft-365-copilot/extensibility/prerequisites#prerequisites)
- [Microsoft 365 Agents Toolkit (ATK)](https://aka.ms/teams-toolkit) VS Code extension 5.0.0+ **or** the [ATK CLI](https://aka.ms/teamsfx-toolkit-cli)
- Ability to **sideload custom apps** in your test tenant (for the sideload/individual-user path)

---

## Install the Microsoft 365 Agents Toolkit (ATK)

You develop, package, and publish this template with ATK. Install it once:

### Option A — VS Code extension (recommended)

1. Install [Visual Studio Code](https://code.visualstudio.com/) and [Node.js 22](https://nodejs.org/).
2. Open VS Code → **Extensions** (`Ctrl+Shift+X`).
3. Search for **"Microsoft 365 Agents Toolkit"** (publisher **TeamsDevApp** — this repo already recommends it in `.vscode/extensions.json`) and select **Install**. It also installs directly from the [Marketplace](https://aka.ms/teams-toolkit).
4. Open **this folder** (`SalesAgentExtensibilitySDLC`) as the workspace root in VS Code. The Agents Toolkit icon appears in the Activity Bar.
5. In the Agents Toolkit panel, **sign in** with your Microsoft 365 test account (and, if prompted, an Azure account).
6. Use the **ENVIRONMENT** section of the panel to pick `dev` / `uat` / `sit`, then run **Provision** / **Publish** from **LIFECYCLE**.

### Option B — ATK CLI (for pipelines / headless)

```bash
npm install -g @microsoft/m365agentstoolkit-cli
atk -h
```

Then use `atk provision --env <env>` / `atk publish --env <env>` as shown below.

---

## Supported extensibility — test ONLY what production accepts

> 🚨 **Production parity is the whole point of this template.** Anything you can technically add here in a lower environment will *run* on the preview agent — but the production Sales agent only accepts a **narrow, supported subset**. If you validate an unsupported pattern here, it will fail when you try to apply it to prod. **Test only the supported patterns below.**

The production Sales agent is extended from the **Custom tools & knowledge** tab in the Microsoft 365 admin center, which copies tools and knowledge from **one** declarative agent. See [Extend Sales agent with custom tools and knowledge](https://learn.microsoft.com/en-us/microsoft-sales-copilot/extend-sales-chat-custom-tools).

| Pattern | Supported in production Sales agent? | Notes |
| --- | --- | --- |
| **API plugins** (OpenAPI-based actions) | ✅ Yes | The primary supported "tool". Retrieve/create/update/delete or run any action exposed by your app's REST API. |
| **MCP server tools** | ✅ Yes | Actions exposed via an MCP server are supported as tools. |
| **Knowledge sources** (SharePoint sites, websites, other supported content) | ✅ Yes | Added as grounding knowledge for the agent. |
| Declarative-agent **capabilities** (e.g. `CodeInterpreter`, `WebSearch`, and others you add) | ✅ Yes — *additively* | Capabilities **can** be added. Add new ones alongside the existing capabilities; confirm each capability you rely on is supported by the production Sales agent before promoting. |
| Anything else (custom runtimes, unsupported connectors, etc.) | ❌ No | Works here, will **not** work in prod. Don't ship it. |

**Production limits to design against (validate them here first):**

- You can copy tools/knowledge from **only one** declarative agent — consolidate everything you want to promote into this single package.
- Combined size of all copied custom tools + knowledge must be **≤ 150 KB**.
- Out-of-the-box Sales tools/knowledge **cannot be removed**; your extensions are additive.
- Ensure all target users have permission to the underlying tools/knowledge.

> ✅ **Rule of thumb — keep every change additive.** Add API plugins, MCP tools, supported knowledge sources, and capabilities *on top of* the base agent. **Do not** edit `appPackage/instruction.txt` and **do not** remove the existing entries in the `capabilities` array of `declarativeAgent.json` (e.g. `CodeInterpreter`). Removing or rewriting base content breaks parity with the production Sales agent — you only ever *add*.

---

## What's in the package

| Path | Purpose |
| --- | --- |
| `appPackage/manifest.json` | Microsoft 365 app manifest. Name and descriptions are **templatized** with `${{…}}` tokens. |
| `appPackage/declarativeAgent.json` | The declarative agent definition (name, description, capabilities, **non-prod disclaimer**). |
| `appPackage/instruction.txt` | Agent instructions (unchanged base Sales behavior). |
| `appPackage/color.png`, `outline.png` | Icons. |
| `env/.env.local` `.env.dev` `.env.uat` `.env.sit` | **One file per environment.** Holds the agent identity (`TEAMS_APP_ID`), version, name suffix, and Dynamics/tool values for that stage. `.env.dev/.uat/.sit` are **committed on purpose** — see [Duplicate agents](#duplicate-agents-in-the-tenant-after-every-deploy). |
| `m365agents.yml`, `m365agents.local.yml` | ATK lifecycle (provision / publish) definitions. |

### The templatized tokens

These `${{TOKEN}}` values are substituted from the active `env/.env.<env>` file when ATK builds the package:

| Token | Set in | Meaning |
| --- | --- | --- |
| `APP_NAME_SUFFIX` | env file | Suffix appended to the agent name, e.g. ` (Dev)`, ` (UAT)`, ` (SIT)`. Keep it short (Teams `name.short` ≤ 30 chars). |
| `TEAMSFX_ENV` | env file | The environment key (`dev`/`uat`/`sit`/`local`). |
| `AGENT_VERSION` | env file | The app package version (`manifest.json` → `version`). **Bump this** to ship a new version of the same agent. |
| `TEAMS_APP_ID` | env file (written back by provision) | **The agent's identity in the tenant.** Provision writes it back into `env/.env.<env>` — **commit it**. See [Duplicate agents](#duplicate-agents-in-the-tenant-after-every-deploy). |

**Optional (opt-in) variables — commented out by default** in each `env/.env.*` file. The base agent does **not** reference them, so packaging works out of the box. Enable them only when *your tools* need them (see [Enabling the Dynamics/tool variables](#enabling-the-dynamicstool-variables)):

| Variable | Meaning |
| --- | --- |
| `DATAVERSE_ENVIRONMENT_NAME` | Friendly name of the non-prod Dynamics environment for this stage. |
| `DATAVERSE_ENVIRONMENT_ID` | The Dynamics/Dataverse **environment ID**. |
| `DATAVERSE_ENVIRONMENT_URL` | The org URL of that environment. |
| `TOOL_BASE_URL` | Base URL your external tools/flows call for this stage. |
| `TOOL_CLIENT_ID` | Auth client id used by your external tools for this stage. |

> ⚠️ **Do not reference a variable that is left commented/empty.** ATK fails packaging with `manifest.MissingEnvironmentVariablesError` if any `${{...}}` token in the manifest or `declarativeAgent.json` has no value. Uncomment and set a variable **before** you reference it.

---

## Update an existing agent instead of creating a new one

Use this when **someone else has already deployed a stage** (dev/uat/sit) and you're touching it for the first time — a new team member, a second developer, a new laptop, or a CI pipeline. Follow these steps and your deploy **updates** the existing agent; skip them and ATK creates a duplicate.

> **The one rule:** `manifest.json` sets `"id": "${{TEAMS_APP_ID}}"`. `teamsApp/create` reuses the existing agent **only** when `TEAMS_APP_ID` already holds an app ID that exists **in the tenant you're signed in to**. Empty ID, or an ID from a different tenant → new app, duplicate agent.

### Step 1 — Get the latest env file

```bash
git pull
cat env/.env.dev          # PowerShell: Get-Content env\.env.dev
```

Look at `TEAMS_APP_ID`:

| What you see | What it means | Go to |
| --- | --- | --- |
| `TEAMS_APP_ID=6f1a…` (a GUID) | The previous deployer committed the identity. **Nothing to recover.** | [Step 3](#step-3--sign-in-to-the-same-tenant) |
| `TEAMS_APP_ID=` (blank) | The identity was never committed. **You must recover it**, or you'll create a duplicate. | [Step 2](#step-2--recover-the-ids-when-the-env-file-is-blank) |

### Step 2 — Recover the IDs when the env file is blank

Pick whichever source you can reach fastest.

**Option A — Ask the person who deployed it (fastest and most reliable).**
Their working copy still has the values ATK wrote back. Ask them to run this in the repo and send you the output:

```bash
git diff env/.env.dev        # shows the values ATK wrote but never committed
```

Better still, ask them to just commit it: `git add env/.env.dev && git commit -m "Pin dev agent identity" && git push`. Then re-run Step 1.

**Option B — Read it from the Developer Portal.**

1. Go to the [Developer Portal](https://dev.teams.microsoft.com/apps) and sign in with an account **in the same tenant** the agent was deployed to.
2. Select **Apps** and find the agent by name — `Sales Agent Preview(Dev)` / `(UAT)` / `(SIT)`. The `APP_NAME_SUFFIX` in each env file tells you which name maps to which stage.
3. Open it. **Overview → Dashboard → Basic information** shows the **App ID** and the **Version**. (**Configure → Basic information** shows the same fields and lets you edit them.)
4. Copy the **App ID** → that is `TEAMS_APP_ID`.
5. Note the **Version** → that is your current `AGENT_VERSION`.

> If several similarly-named apps are listed, you're already looking at the duplicates. Pick the one that is actually installed/published — check **Publish to org** status or the newest **Version** — and clean up the rest afterwards.

**Option C — Get the M365 title values (optional).**
`M365_TITLE_ID`, `M365_APP_ID`, and `SHARE_LINK` are convenience outputs; provision regenerates them. If you want them up front, ask the original deployer for their env file, or resolve them from the app ID:

```bash
atk launchinfo --manifest-id <TEAMS_APP_ID>
```

### Step 3 — Sign in to the same tenant

The app ID only resolves inside the tenant it was created in. A valid ID + the wrong tenant still produces a duplicate.

- **VS Code:** Agents Toolkit → **ACCOUNTS** → confirm the signed-in M365 account. If several accounts are listed, sign out of the wrong ones — provision and publish must both use the target tenant.
- **CLI:** `atk auth --help` to list the account commands, then sign in with the correct M365 account.

### Step 4 — Make sure you're allowed to update the app

An app in the Developer Portal has **owners**. If you aren't one, your provision can fail or silently push you toward creating your own copy.

- The original deployer can add you in VS Code: Agents Toolkit → **ENVIRONMENT** → **Manage Collaborators** → **Add App Owners**, then enter your M365 account email.
- Or in the [Developer Portal](https://dev.teams.microsoft.com/apps): open the app → **Advanced** → **Owners** → **Add owners** → pick your user → **Role** = **Administrator** (can add/remove owners and delete) or **Operative** (can update configuration) → **Add**.
- If the app has **no active owners** left (for example, the original deployer left the org), a tenant admin can claim it by entering the app ID in the Developer Portal.

### Step 5 — Fill in the env file and commit it

Put the recovered values into the committed `env/.env.<env>` file — **not** `env/.env.<env>.user`, which is gitignored and only for `SECRET_*` values.

```dotenv
# env/.env.dev
TEAMS_APP_ID=6f1a2b3c-4d5e-6f70-8192-a3b4c5d6e7f8   # from Step 2 — the agent's identity, never changes
AGENT_VERSION=1.2.0                                  # current version from the portal, bumped by one
M365_TITLE_ID=                                       # optional — provision fills these in
M365_APP_ID=
SHARE_LINK=
```

Then commit it so the next person doesn't repeat this recovery:

```bash
git add env/.env.dev
git commit -m "Pin dev agent identity (TEAMS_APP_ID)"
git push
```

### Step 6 — Bump the version

Set `AGENT_VERSION` **higher than the version currently shown in the Developer Portal**. An equal or lower version means the update isn't picked up as a new revision.

| Portal shows | Set `AGENT_VERSION` to |
| --- | --- |
| `1.0.0` | `1.0.1` (fix) or `1.1.0` (new tool/knowledge) |
| `1.2.0` | `1.2.1` / `1.3.0` |

`TEAMS_APP_ID` stays **exactly** as it is. Version = which revision; app ID = which agent.

### Step 7 — Provision and publish

```bash
atk provision --env dev
atk publish --env dev
```

Or in VS Code: pick the env, then **LIFECYCLE → Provision**, then **Publish to Organization**. Always in that order.

### Step 8 — Verify you updated rather than duplicated

1. Open the [Developer Portal](https://dev.teams.microsoft.com/apps) → **Apps**. There should still be **exactly one** `Sales Agent Preview(<ENV>)` — no new entry.
2. Open it → **Overview → Basic information**. The **App ID** must match the `TEAMS_APP_ID` in your env file, and the **Version** must be the `AGENT_VERSION` you just set.
3. Check `git diff env/.env.dev`. If `TEAMS_APP_ID` changed, ATK created a new app — stop, restore the original ID, and clean up the extra app.

### Pre-flight checklist

Run through this before every provision on a shared stage:

- [ ] `git pull` done — you have the latest env file.
- [ ] `TEAMS_APP_ID` is **non-empty** in `env/.env.<env>`.
- [ ] The signed-in M365 account is in the **same tenant** the agent lives in.
- [ ] You're an **owner** of the app in the Developer Portal.
- [ ] `AGENT_VERSION` is higher than the version in the portal.
- [ ] After provision: `git diff` shows `TEAMS_APP_ID` **unchanged**, and any newly written IDs are committed.

### If you're the first person to deploy a stage

There's nothing to recover — but you own the handoff:

1. Leave `TEAMS_APP_ID=` blank and run `atk provision --env <env>`. ATK creates the app and writes the ID into `env/.env.<env>`.
2. **Commit `env/.env.<env>` immediately** — including `TEAMS_APP_ID`, `M365_TITLE_ID`, `M365_APP_ID`, and `SHARE_LINK`.
3. Add your teammates as app owners (Step 4) so they can update it.

Everyone after you then lands in the easy path at Step 1. Skipping step 2 is the single most common cause of [duplicate agents](#duplicate-agents-in-the-tenant-after-every-deploy).

---

## Quick start

> 👥 **Not the first person on this project?** If a teammate has already deployed the stage you're targeting, do [Update an existing agent instead of creating a new one](#update-an-existing-agent-instead-of-creating-a-new-one) **before** you run Provision — otherwise you'll create a duplicate agent in the tenant.

### 1. Manage your environments with `env/.env.*`

Open the env file for the stage you're building (start with `env/.env.dev`). The only value you must set is the name suffix — the Dynamics/tool variables are **optional and commented out** by default:

```dotenv
APP_NAME_SUFFIX= (Dev)
```

Do the same in `env/.env.uat` and `env/.env.sit`. Each stage produces a **separate, isolated agent** named `Sales Agent Preview (Dev)`, `Sales Agent Preview (UAT)`, `Sales Agent Preview (SIT)`.

> To add a brand-new stage (e.g. `preprod`), copy an existing `env/.env.*` file to `env/.env.preprod` and adjust the values.

#### Enabling the Dynamics/tool variables

The base agent ships with no Dynamics dependency, so it packages immediately. When you add a tool that needs to target a specific non-production Dynamics environment or backend:

1. Open the relevant `env/.env.<env>` file and **uncomment + fill** the variables you need, e.g.:
   ```dotenv
   DATAVERSE_ENVIRONMENT_ID=00000000-0000-0000-0000-000000000000
   DATAVERSE_ENVIRONMENT_URL=https://contoso-dev.crm.dynamics.com
   TOOL_BASE_URL=https://dev.api.contoso.com
   TOOL_CLIENT_ID=<dev-app-registration-client-id>
   ```
2. Reference them **only from the tool/plugin manifest that consumes them** (e.g. your OpenAPI `server.url` or MCP endpoint) as `${{TOOL_BASE_URL}}`.
3. Set the same-named variable (with the matching value) in every env file you build, so no stage packages with an empty reference.

### 2. Add your external tools with ATK

This template ships with only the base agent (plus `CodeInterpreter`). Add your own extensibility with the Agent Toolkit — but stick to the [supported patterns](#supported-extensibility--test-only-what-production-accepts) (API plugins, MCP tools, knowledge sources) so what you validate here will actually transfer to production:

- **API plugins / actions** — add an action to `declarativeAgent.json` and reference your OpenAPI/plugin manifest. Point the tool's server URL at your per-stage backend using the env tokens, e.g. `${{TOOL_BASE_URL}}`.
- **MCP / connected agents / knowledge** — add the corresponding capability or action and target the matching non-prod environment.

Use the ATK command palette (**Microsoft 365 Agents Toolkit: Add …**) to scaffold plugins, then wire their environment-specific values to `${{TOOL_BASE_URL}}` / `${{TOOL_CLIENT_ID}}` / `${{DATAVERSE_ENVIRONMENT_URL}}` so the same source produces the right package for each stage.

### 3. Provision / build for a specific environment

Using the ATK CLI:

```bash
# Build + provision the Dev agent
atk provision --env dev

# Later, build UAT / SIT the same way
atk provision --env uat
atk provision --env sit
```

Or in VS Code: pick the environment, then **Provision**. The built package lands in `appPackage/build/appPackage.<env>.zip`.

### 4. Test the extension — sideload or publish

> ⚠️ **Always Provision *before* you Publish, per environment.** In VS Code the **LIFECYCLE** panel lists **Provision** above **Publish to Organization** — run them top-to-bottom. **Provision** (`teamsApp/create`) creates the app in the tenant and writes a valid `TEAMS_APP_ID` into `env/.env.<env>`; **Publish** only *updates* an app that already exists. Publishing without a successful Provision in the **currently signed-in tenant** fails with `TeamsAppNotExists` (see [Troubleshooting](#troubleshooting)).

- **Sideload to an individual user (fastest inner loop):** upload `appPackage/build/appPackage.<env>.zip` via Copilot/Teams **Upload a custom app**. Only that user gets the preview agent — ideal for a developer validating a tool without any admin or overlay step. (Requires custom-app upload to be enabled.)
- **Publish to your test tenant (shared testing):** run **Provision** first, then the publish flow (`atk publish --env <env>` or **Publish to Organization** in VS Code). An admin approves it in the Microsoft 365 admin center, and the isolated preview agent becomes available to your test group. `AGENT_SCOPE` in the env file controls the provision scope (`shared` for sideload-style testing).

After a successful provision + publish you'll see the agent as **`Sales Agent Preview(<ENV>)`** in the [Developer Portal](https://dev.teams.microsoft.com/apps) and be able to open it in [Microsoft 365 Copilot](https://m365.cloud.microsoft/chat/). Provision writes `TEAMS_APP_ID`, `M365_TITLE_ID`, `M365_APP_ID`, and `SHARE_LINK` back into **`env/.env.<env>`** (only `SECRET_*` variables go to `env/.env.<env>.user`).

> 🔁 **Commit `env/.env.<env>` right after the first successful provision of a stage.** `TEAMS_APP_ID` is the agent's identity — if it is blank on the next run, ATK creates a *second* agent instead of updating the first. See [Duplicate agents in the tenant after every deploy](#duplicate-agents-in-the-tenant-after-every-deploy).

### 4b. Ship a new version of the same agent

Bump `AGENT_VERSION` in `env/.env.<env>` (for example `1.0.0` → `1.1.0`) and re-run Provision (then Publish). **Leave `TEAMS_APP_ID` exactly as it is** — the version is a property of the app, the app ID is its identity.

```dotenv
# env/.env.dev
TEAMS_APP_ID=00000000-0000-0000-0000-000000000000   # never changes for this stage
AGENT_VERSION=1.1.0                                  # the only thing you bump
```

### 5. Promote

Once your tools validate on the preview agent for a stage, promote the **same tool configuration** to the next stage's env file (or, finally, apply the equivalent overlay to the **production** Sales agent). The preview agents themselves are never promoted — they exist only to validate the extensions.

---

## Evaluating the agent (optional)

Install the M365 Copilot Agent Evaluations CLI to score your agent's quality. Requires [admin consent](https://github.com/microsoft/work-iq/blob/main/ADMIN-INSTRUCTIONS.md) at tenant level.

1. `npm install -g @microsoft/m365-copilot-eval`
2. Fill the `AZURE_AI_*` variables in the env file ([how to get them](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/evaluations-cli-get-env-values#get-your-azure-openai-endpoint-and-api-key)).
3. Provision the target env first, then run `runevals --env dev`.

A sample dataset lives in `evals/prompts.json`. [Read more](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/evaluations-cli-overview).

---

## Guardrails

- **All changes must be additive.** Only *add* tools, actions, knowledge, and capabilities — never remove or rewrite what the base agent ships with.
- **Do not edit `appPackage/instruction.txt`.** It carries the base Sales agent behavior; keep it unchanged so the preview stays faithful to production.
- **Do not remove existing entries from the `capabilities` array** in `declarativeAgent.json` (e.g. `CodeInterpreter`). Add new capabilities alongside them.
- **Never** flip this package to your production Sales agent, and keep the non-prod disclaimer intact.
- Keep `env/.env.*` pointed at **non-production** Dynamics environments and app registrations only.
- Keep `APP_NAME_SUFFIX` short and environment-distinct so the isolated agents are easy to tell apart.

## Troubleshooting

### Duplicate agents in the tenant after every deploy

**Symptom:** Every time you bump the version and re-run Provision/Publish, a *new* `Sales Agent Preview(<ENV>)` entry shows up in the tenant instead of the existing one being updated. Over time the Developer Portal / app catalog fills with near-identical agents.

**Cause:** `TEAMS_APP_ID` was empty (or stale) when `teamsApp/create` ran.

`manifest.json` sets `"id": "${{TEAMS_APP_ID}}"`, so **`TEAMS_APP_ID` is the agent's identity in the tenant**. The `teamsApp/create` action's documented behavior is:

> If the environment variable that stores the Teams app ID is empty or the app ID isn't found from the Teams Developer Portal, then this action creates a new Teams app.

So `teamsApp/create` only *reuses* the agent when `TEAMS_APP_ID` already holds a valid ID for the **currently signed-in tenant**. Anything that returns that variable to blank makes the next provision mint a brand-new app — and the previous one stays behind as a duplicate. The usual culprits:

| Trigger | Why it blanks the ID |
| --- | --- |
| **CI/CD pipeline** | A fresh `actions/checkout` gets the committed `env/.env.<env>`. If the ID was never committed, every pipeline run starts from `TEAMS_APP_ID=` and creates a new agent. |
| **Fresh clone / new machine / another teammate** | Same as above — the write-back from someone else's provision only lived in their working copy. |
| **`git checkout` / `git stash` / discarding changes** | Provision's write-back to `env/.env.<env>` is an *uncommitted* file change. Reverting the file throws the ID away. |
| **Regenerating or hand-resetting the env file to "start clean" before a version bump** | Blanks the identity along with everything else. |
| **Provisioning against a different tenant/account** | The ID is valid but not found in *this* tenant, so a new app is created there. |

Note that bumping the version is not itself the cause — it's just when you notice. The version (`AGENT_VERSION`) is a property of the app; `TEAMS_APP_ID` is *which* app.

**Fix:**

> Joining a stage someone else already deployed? Follow [Update an existing agent instead of creating a new one](#update-an-existing-agent-instead-of-creating-a-new-one) for the full step-by-step recovery.

1. Pick the one agent you want to keep for the stage. Get its **App ID** from the [Developer Portal](https://dev.teams.microsoft.com/apps) (or from the working copy where provision last succeeded).
2. Write it into the committed env file and **commit it**:
   ```dotenv
   # env/.env.dev
   TEAMS_APP_ID=00000000-0000-0000-0000-000000000000
   AGENT_VERSION=1.0.0
   ```
   Do the same for `env/.env.uat` and `env/.env.sit`, each with **its own distinct ID** — one identity per stage.
3. Let the first successful provision of a stage also write back `M365_TITLE_ID`, `M365_APP_ID`, and `SHARE_LINK` into the same file, and commit those too.
4. To ship a change from then on: bump **`AGENT_VERSION` only**, re-run Provision → Publish. The existing agent is updated in place.
5. Delete the leftover duplicates from the Developer Portal / Microsoft 365 admin center.

**Rules to keep it fixed:**

- `env/.env.<env>` is tracked in git **on purpose**. Commit the write-back; never `git checkout` it away.
- Only `SECRET_*` variables belong in the gitignored `env/.env.<env>.user`. `TEAMS_APP_ID` is not a secret.
- In CI, either commit the IDs or inject `TEAMS_APP_ID` per stage from a pipeline variable — never let a pipeline run with it blank.
- Every stage keeps its **own** `TEAMS_APP_ID`. Reusing one ID across dev/uat/sit makes the stages overwrite each other instead of coexisting.
- Any new env file you add (`.env.preprod`, and the auto-generated `.env.local`) must define `TEAMS_APP_ID` and `AGENT_VERSION`, or packaging fails with `manifest.MissingEnvironmentVariablesError`.

> Related: `provision` runs `copilotAgent/publish` with `scope: ${{AGENT_SCOPE}}` (`shared`) and `publish` runs it again with `scope: tenant`, writing to `M365_TITLE_ID` and `M365_PUBLISHED_TITLE_ID` respectively. That is expected — both point at the *same* Teams app as long as `TEAMS_APP_ID` is stable. If they diverge, you provisioned and published with different app IDs.

### `TeamsAppNotExists` — "App with ID … does not exist in Developer Portal" when publishing

```
Failed to Execute lifecycle publish due to failed action: teamsApp/update.
TeamsAppNotExists: App with ID <guid> does not exist in Developer Portal.
```

**Cause:** You ran **Publish to Organization** without a successful **Provision** in the currently signed-in tenant. Only the `provision` lifecycle contains `teamsApp/create`; `publish` just runs `teamsApp/update`, which requires the app (identified by `TEAMS_APP_ID`) to already exist. The ID it referenced was never created in this tenant, was deleted, or belongs to a different account/tenant.

**Fix:**
1. In the ATK panel, confirm the **correct M365 account/tenant is active** (this template may have several accounts signed in — provision and publish must use the same one).
2. Run **LIFECYCLE → Provision** for the target env (e.g. `sit`). This creates the app and writes a fresh `TEAMS_APP_ID` to `env/.env.<env>`.
3. Run **LIFECYCLE → Publish to Organization** for the same env.
4. If `env/.env.<env>` holds a **stale** `TEAMS_APP_ID` (deleted app or wrong tenant), set it back to `TEAMS_APP_ID=` and re-run Provision so a new app is created — then **commit the new ID**. ⚠️ Do this only when the app is genuinely gone; blanking a *valid* ID is what causes [duplicate agents](#duplicate-agents-in-the-tenant-after-every-deploy).

### Warning: "Short name contains Beta environment keywords (STAG/Staging/Preview)"

This is an **informational validation warning, not an error** (packaging still succeeds — e.g. `1 warning, 60 passed`). It fires because the agent is deliberately named `Sales Agent Preview…` to keep it clearly non-production. Leave it as-is for lower environments — the naming is intentional so the preview agents are never mistaken for the production Sales agent.

### `manifest.MissingEnvironmentVariablesError` during packaging

A `${{…}}` token in `manifest.json` / `declarativeAgent.json` (or a tool manifest) references a variable that is empty or still commented out in `env/.env.<env>`. Uncomment and set the variable **before** you reference it, and set it in **every** env file you build. See [Enabling the Dynamics/tool variables](#enabling-the-dynamicstool-variables).

### Wrong tenant / agent not showing up

Provision, publish, and the browser session where you open Copilot must all be the **same tenant**. Switch the active account in the ATK **ACCOUNTS** panel, then re-provision. Verify the app in the [Developer Portal](https://dev.teams.microsoft.com/apps) and open it via the `SHARE_LINK` from `env/.env.<env>`.

---

## References

- [Declarative agent schema 1.8 for Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/declarative-agent-manifest-1.8)
- [Build declarative agents with the Agents Toolkit](https://aka.ms/teams-toolkit-declarative-agent)
- [Extend the Sales agent with custom tools and knowledge (production)](https://learn.microsoft.com/en-us/microsoft-sales-copilot/extend-sales-chat-custom-tools)
- [API plugins for declarative agents](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/overview-api-plugins)
- [Knowledge sources for declarative agents](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/knowledge-sources)
