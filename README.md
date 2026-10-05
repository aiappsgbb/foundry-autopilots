# Foundry autopilots

An autopilot is an AI colleague in Microsoft 365. It has its own account, mailbox, Teams presence and manager.
People @mention it in group chats, CC it on email and assign it comments in Word, and it does the work as
itself, under rules an administrator approved. It runs as a Microsoft Foundry hosted agent and is hired through
Agent 365.

This repo is one skill that takes you, or your coding agent, from nothing to a working colleague:

1. **First reply.** Deploy Microsoft's
   [autopilot sample](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/foundry-autopilot-agent),
   get it approved, hire it in Teams and say hi.
2. **A real colleague.** Swap the sample's single model call for the GitHub Copilot SDK. Put the Agent Governance
   Toolkit in front of every tool call. Add role skills, a memory store, an email filter and versioned releases.

We wrote it after building two of these, a compliance officer and a procurement officer, so it includes what the
docs don't cover yet: the licensing and approval steps that stall most first attempts, the channel quirks, and
which design choices mattered and which were dead ends.

## What's in the skill

| File | What it covers |
| --- | --- |
| [SKILL.md](skills/setting-up-foundry-autopilots/SKILL.md) | The steps, each with a "done when" check, rules for the agent running it, model choice, and a troubleshooting table |
| [concepts.md](skills/setting-up-foundry-autopilots/concepts.md) | What an autopilot is, when to build one rather than an ordinary agent, and use cases that fit |
| [lessons.md](skills/setting-up-foundry-autopilots/lessons.md) | Lessons for anyone building an autopilot harness: hiring, harness design, channels, memory, governance, models, release |
| [reference.md](skills/setting-up-foundry-autopilots/reference.md) | Exact commands, request bodies and code |

## Use it

Copy `skills/setting-up-foundry-autopilots/` into your agent's skills folder (for GitHub Copilot CLI,
`~/.copilot/skills/`) and ask: *"Help me set up a Foundry autopilot."* The skill tells the agent to confirm
prerequisites first and to ask before creating anything in your tenant. You can also just read it.

Foundry hosted agents, Agent 365 and the APIs used here are in preview and change often. Everything was checked
in October 2026.

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

## License

[MIT](LICENSE). Built by the Microsoft GBB team.
