# Reference

Values in `<angle brackets>` are yours. The Foundry agent, skills and memory APIs are in preview; the API versions
are the ones that worked for us in October 2026. Setup for the commands below:

```bash
EP=<project endpoint, https://<account>.services.ai.azure.com/api/projects/<project>>
AGENT=<agent name>
TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)
```

## Stage A details

**What `azd provision` does.** The sample's `preprovision` hook registers resource providers. Bicep creates a
Foundry account and project, a model deployment, a container registry, and Application Insights. The
`postprovision` hook then creates a Foundry toolbox (web search, code interpreter), builds the image in the
registry (`az acr build`), creates the agent version, and publishes it. To rerun only the hooks after a change, run
`azd provision` again: each run creates a new agent version.

**The publish request** (`scripts/publish-digital-worker.ps1`). These fields make it an autopilot rather than a
store agent:

| Field | Value | Why |
| --- | --- | --- |
| `publishAsAutopilot` | `true` | A blueprint that hires instances |
| `publishScope` | `"Tenant"` | Goes to your administrator for approval |
| `agentDisplayName`, descriptions, developer and privacy URLs | Placeholders in the sample | What the admin and the people hiring it see. Set your own before the first publish |
| `optionalPermissionScopes` | Agent 365 Tools (`ea9ffc3e-8a23-4a7d-836d-234d7c7565c1`): `McpServers.Mail.All`, `.Teams.All`, `.Word.All`, `.Calendar.All`, `.OneDriveSharepoint.All`, `.Excel.All`; plus Azure DevOps (`2a72489c-aab2-4b65-b93a-a91edccf33b8`): `Ado.Mcp.Tools` | What the admin consents to in the approval wizard. Microsoft Learn's quickstart doesn't list this field, and without it the agent can't use the Microsoft 365 tools. Drop the Azure DevOps entry if you don't need it |
| `canRespondWithoutMention` | `true` in the sample | `false` makes it answer only @mentions in group chats and channels |
| `appVersion` | `"1.0.0"` in the sample | The script ignores `version already exists`, so rerunning `azd provision` builds a new agent version but **doesn't republish**. To change any field in this table, raise `appVersion` in the script first, then have the admin approve the update |

**Session logs.** Every failed reply returns a session ID. Stream that session's container log:

```bash
curl -N -H "Authorization: Bearer $TOKEN" -H "Accept: text/event-stream" -H "Foundry-Features: HostedAgents=V1Preview" \
  "$EP/agents/$AGENT/sessions/<session-id>:logstream?api-version=2025-11-15-preview"
```

**Microsoft 365 tokens.** The sample doesn't need the hired instance's IDs. For each incoming message it exchanges a
token for the Agent 365 MCP server's scope through the Microsoft 365 Agents SDK's authorization handler
(`auth.exchange_token(context, scopes=[...], auth_handler_id=...)`), and sends that token to the MCP server. Keep
this pattern when you change the brain.

## B1. The GitHub Copilot SDK as the brain

**Where it goes in the sample.** The code is in `src/hello_world_a365_agent/`. In `agent.py`, the Teams, email and
Word-comment handlers all call `_invoke_responses_api`, the one model call. Replace that method's body with a
Copilot SDK session and leave the handlers alone. Keep `_acquire_mcp_token`: it gives you the per-turn token for the
Agent 365 MCP servers, which you pass to the session's `mcp_servers`. The image builds from
`src/hello_world_a365_agent/foundry-infra/Dockerfile`; add the SDK there.

Python package `github-copilot-sdk`. In the Dockerfile, after `pip install`, run `python -m copilot download-runtime`.
Start one `CopilotClient` per container. Create one session per conversation (derive a stable session ID from the
conversation ID), with the model behind your Azure OpenAI deployment. This worked with SDK 1.0.14. Names in
lower case without a definition are yours to supply:

```python
from copilot import CopilotClient, ProviderConfig

session = await client.create_session(
    session_id=session_id,                                  # stable per conversation
    model="<deployment name>",
    provider=ProviderConfig(
        type="openai", base_url="https://<account>.openai.azure.com/openai/v1/", wire_api="responses",
        bearer_token_provider=token_for_cognitiveservices,  # managed identity, https://cognitiveservices.azure.com/.default
    ),
    tools=host_tools,                                       # your own tools: save a redline, remember, send one email
    mcp_servers={"m365-mail": {"type": "http", "url": "https://agent365.svc.cloud.microsoft/agents/servers/mcp_MailTools",
                               "tools": ["*"], "headers": {"Authorization": "Bearer " + mcp_token}}},  # from _acquire_mcp_token
    enable_skills=True, skill_directories=[skills_dir],     # B3
    custom_agents=sub_agents,                               # e.g. one "item-worker" per separate ask, run in parallel
    infinite_sessions={"enabled": True},                    # background context compaction
    on_permission_request=permission_handler,               # B2
    request_extensions=True, extension_sdk_path=agt_sdk_dir, hooks={"on_pre_tool_use": agt_gate},  # B2
    working_directory=work_dir, enable_config_discovery=False,
)
await session.send(inbox_item_text)                         # then read events until the session is idle
```

The runtime names MCP tools `<label>-<tool>` (for example `m365-mail-SendEmailWithAttachments`). Your governance
policy matches those names.

## B2. The Agent Governance Toolkit

AGT runs as a Copilot runtime extension. Its `preToolUse` hook allows, denies or asks for review on every call,
including MCP tools, sub-agents and tools the runtime would otherwise auto-approve.

Install it into the Copilot home the SDK uses (Node 22 or later). Pin `@github/copilot-sdk` to the same version as
the Python package:

```bash
npm install --save-exact @microsoft/agent-governance-copilot-cli@5.0.0 @github/copilot-sdk@1.0.14
node node_modules/@microsoft/agent-governance-copilot-cli/bin/agt-copilot.mjs install --copilot-home <home>
cp policy.json <home>/agt/policy.json     # the extension reads AGT_COPILOT_POLICY_PATH; audit goes to AGT_COPILOT_AUDIT_PATH
```

A minimal policy. Tools in `allowedTools` run, `blockedToolCalls` deny with a named reason that appears in the log,
and anything else gets the default effect:

```json
{
  "schemaVersion": 1, "version": 1, "mode": "advisory", "denyOnPolicyError": true,
  "toolPolicies": {"defaultEffect": "review", "allowedTools": ["skill", "task", "m365-mail-GetMessage", "send_email"]},
  "blockedToolCalls": [
    {"id": "no-deletions", "tool": "m365-mail-DeleteMessage", "effect": "deny",
     "reason": "no-deletions: the agent never deletes mail.", "commandPatterns": [{"source": "[\\s\\S]"}]},
    {"id": "inside-only", "tool": "send_email", "effect": "deny",
     "reason": "inside-only: email goes to company addresses only.",
     "commandPatterns": [{"source": "[A-Za-z0-9._%+'-]+@(?!contoso\\.com(?![A-Za-z0-9.-]))[A-Za-z0-9-]"}]}
  ]
}
```

What we learned wiring it into a hosted container:

- **Launching the extension.** A standalone runtime has no extension launcher. SDK 1.0.14 had no public API for it,
  so we registered a handler for `extensionLaunchProvider.resolve` that starts only the AGT extension with `node` and
  passes `AGT_COPILOT_POLICY_PATH` and `AGT_COPILOT_AUDIT_PATH`. Check for a public API in your SDK version first.
- **Fail closed.** Add an `on_pre_tool_use` hook that denies every call unless the AGT extension's status is
  `running`. In the permission handler, reject requests of kind `hook`: that's AGT saying "review", and nobody is
  there to review.
- **Arguments.** AGT pattern-matches top-level string arguments only. Recipients passed as arrays are invisible to it,
  so deny the Agent 365 mail-sending tools and give the agent its own `send_email(to, subject, body)` tool with a
  single `to` string that a rule can check.
- **Modes.** In `enforce` mode AGT's heuristic text scanner blocked ordinary compliance wording ("override", "you
  must"). `advisory` keeps every rule above deterministic; only the heuristic scanners report instead of block.
- **Prompt text.** The extension adds its own prompt text. With it, our agent stopped delegating to sub-agents;
  removing that text from the installed extension fixed it without changing any decision.
- **Test the policy** with AGT's own engine before you build: load `lib/policy.mjs` from the installed extension and
  call `evaluatePreToolUse` for each tool call you expect.

## B3. Foundry Skills

Each skill is a folder with a `SKILL.md` (front matter `name` equal to the folder name, plus `description`). The
instance identity needs **Foundry User** on the Foundry resource. Header `Foundry-Features: Skills=V1Preview`,
`api-version=v1`:

| Do | Call |
| --- | --- |
| Upload a version | `POST $EP/skills/<name>/versions`, multipart field `files` = the folder as a zip |
| Make it the default | `POST $EP/skills/<name>` with `{"default_version": "<n>"}` |
| List | `GET $EP/skills?limit=100` |
| Download | `GET $EP/skills/<name>/content` with `Accept: application/zip` |

In the container, at most once a minute, download each skill's default version into the directory you pass as
`skill_directories`. Unpack into a staging folder first, then rename, so the runtime never sees half a skill.

## B4. Foundry memory store

Create it once (header `Foundry-Features: MemoryStores=V1Preview`, `api-version=v1`):

```json
POST $EP/memory_stores
{"name": "<agent>-memory",
 "definition": {"kind": "default", "chat_model": "<chat deployment>", "embedding_model": "text-embedding-3-small",
   "options": {"user_profile_enabled": true, "chat_summary_enabled": true, "procedural_memory_enabled": true,
     "default_ttl_seconds": 31536000,
     "user_profile_details": "Work context only: role, requests, decisions and commitments. Never store health, family, financial or precise location details, or credentials."}}}
```

Items: `POST $EP/memory_stores/<name>/items` with `{"scope", "content", "kind"}`; `POST .../items:list` with
`{"scope"}`; `POST .../:search_memories`; `DELETE .../items/<id>`. Use scope `person-<Entra object ID>` for what
the agent knows about someone, and `<agent>` for its own open items. Build the remember and recall tools per
conversation, bound to the authenticated sender, so the model can never pass a scope. Load person memories only in
one-to-one conversations, so nothing private comes up in a group chat.

## B5. Versions, release and rollback

Header `Foundry-Features: DigitalWorker=V1Preview`, `api-version=2025-11-15-preview`.

- **New version:** `POST $EP/agents/$AGENT/versions` with the full `definition`: image, environment variables and
  `container_protocol_versions` (see the body in the sample's `scripts/agent-creation-script.ps1`). The sample
  uses the `:latest` tag. Use a unique tag or the digest instead, so that rolling back really runs the old code. Add
  `{"protocol": "invocations", "version": "2.0.0"}` beside `activity_protocol` if you want scheduled or on-demand runs.
- **Release:** `PATCH $EP/agents/$AGENT` (`Content-Type: application/merge-patch+json`) with
  `agent_endpoint.version_selector.version_selection_rules = [{"type": "FixedRatio", "agent_version": "<n>",
  "traffic_percentage": 100}]`. Send the current `protocol_configuration` and `authorization_schemes` back
  unchanged, and read the agent back to check them.
- **Rollback:** the same call with the previous version. Sessions bound to the old version keep running it until you
  delete them (`DELETE $EP/agents/$AGENT/endpoint/sessions/<id>`).
- **Behaviour changes** don't need a version: upload a new skill version and make it the default (B3).
- **Read what happened:** stream the session log (Stage A). Log tool names, AGT decisions and timings, never
  message content.

## Removing everything

In the Microsoft 365 admin center, delete each instance (frees its seats) and retire the blueprint. Then `azd down`
in the sample folder. Deleting the Azure resources doesn't remove hired instances.
