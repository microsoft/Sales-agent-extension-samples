# Sales Agent Preview — Extensibility SDLC Template

A **parameterized, non-production** Microsoft 365 declarative-agent template that mirrors the base Sales agent so teams can **develop and validate extensibility** (custom tools, actions, and knowledge) through a proper software development lifecycle — **Dev → UAT → SIT** — *before* the same extensions are applied to the **production** Sales agent.

> ⚠️ **This is a TEST / NON-PRODUCTION agent.** Never publish it as your production Sales agent and never point it at real customer/production data. The agent renders a non-prod disclaimer at the start of every conversation.

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
| `env/.env.local` `.env.dev` `.env.uat` `.env.sit` | **One file per environment.** Holds the name suffix and Dynamics/tool values for that stage. |
| `m365agents.yml`, `m365agents.local.yml` | ATK lifecycle (provision / publish) definitions. |

### The templatized tokens

These `${{TOKEN}}` values are substituted from the active `env/.env.<env>` file when ATK builds the package:

| Token | Set in | Meaning |
| --- | --- | --- |
| `APP_NAME_SUFFIX` | env file | Suffix appended to the agent name, e.g. ` (Dev)`, ` (UAT)`, ` (SIT)`. Keep it short (Teams `name.short` ≤ 30 chars). |
| `TEAMSFX_ENV` | env file | The environment key (`dev`/`uat`/`sit`/`local`). |
| `TEAMS_APP_ID` | generated | Written back by ATK during provision. |

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

## Quick start

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

After a successful provision + publish you'll see the agent as **`Sales Agent Preview(<ENV>)`** in the [Developer Portal](https://dev.teams.microsoft.com/apps) and be able to open it in [Microsoft 365 Copilot](https://m365.cloud.microsoft/chat/). The provision output writes `TEAMS_APP_ID`, `M365_TITLE_ID`, `M365_APP_ID`, and a `SHARE_LINK` into `env/.env.<env>.user`.

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
4. If `env/.env.<env>` holds a **stale** `TEAMS_APP_ID` (deleted app or wrong tenant), set it back to `TEAMS_APP_ID=` and re-run Provision so a new app is created.

### Warning: "Short name contains Beta environment keywords (STAG/Staging/Preview)"

This is an **informational validation warning, not an error** (packaging still succeeds — e.g. `1 warning, 60 passed`). It fires because the agent is deliberately named `Sales Agent Preview…` to keep it clearly non-production. Leave it as-is for lower environments — the naming is intentional so the preview agents are never mistaken for the production Sales agent.

### `manifest.MissingEnvironmentVariablesError` during packaging

A `${{…}}` token in `manifest.json` / `declarativeAgent.json` (or a tool manifest) references a variable that is empty or still commented out in `env/.env.<env>`. Uncomment and set the variable **before** you reference it, and set it in **every** env file you build. See [Enabling the Dynamics/tool variables](#enabling-the-dynamicstool-variables).

### Wrong tenant / agent not showing up

Provision, publish, and the browser session where you open Copilot must all be the **same tenant**. Switch the active account in the ATK **ACCOUNTS** panel, then re-provision. Verify the app in the [Developer Portal](https://dev.teams.microsoft.com/apps) and open it via the `SHARE_LINK` from `env/.env.<env>.user`.

---

## References

- [Declarative agent schema 1.8 for Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/declarative-agent-manifest-1.8)
- [Build declarative agents with the Agents Toolkit](https://aka.ms/teams-toolkit-declarative-agent)
- [Extend the Sales agent with custom tools and knowledge (production)](https://learn.microsoft.com/en-us/microsoft-sales-copilot/extend-sales-chat-custom-tools)
- [API plugins for declarative agents](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/overview-api-plugins)
- [Knowledge sources for declarative agents](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/knowledge-sources)
