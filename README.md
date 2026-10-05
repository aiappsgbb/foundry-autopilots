# Foundry autopilots

A skill that walks you through building an **autopilot**: an AI colleague in Microsoft 365 with its own account,
mailbox, Teams presence and manager. It runs as a Microsoft Foundry hosted agent and is hired through Agent 365.

The skill has two stages:

1. **First reply.** Deploy Microsoft's
   [autopilot sample](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/foundry-autopilot-agent),
   get it approved, hire it in Teams and say hi.
2. **A real colleague.** Replace the sample's model call with the GitHub Copilot SDK, govern every tool call with
   the Agent Governance Toolkit, and add versioned skills, a memory store, an email filter and safe releases.

It also covers the prerequisites that stop most first attempts (Frontier, licences and seats, admin roles) and the
problems we hit along the way.

## Use it

Copy `skills/setting-up-foundry-autopilots/` to your agent's skills folder (for GitHub Copilot CLI,
`~/.copilot/skills/`), then ask: *"Help me set up a Foundry autopilot."* Or read
[SKILL.md](skills/setting-up-foundry-autopilots/SKILL.md) and
[reference.md](skills/setting-up-foundry-autopilots/reference.md) yourself.

Foundry hosted agents, Agent 365 and the APIs used here are in preview and change often. The commands were checked
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
