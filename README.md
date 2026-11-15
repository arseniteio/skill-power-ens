# skill-power-ens

An agent skill that installs OSCompliance ENS and Power ENS on Wazuh 4.14.7 to 4.14.9, audits Linux machines against the ENS, and brings them to compliance through configuration only. The instructions are in [SKILL.md](SKILL.md). The skill follows the documentation of [oscompliance-ens](https://github.com/arseniteio/oscompliance-ens) and [power-ens](https://github.com/arseniteio/power-ens).

The agent installs with the release `install.sh`, or with the verified package when `curl | bash` is not allowed. It changes only the Power ENS configuration, per-control exclusions with a reason, and the agent and group configuration files the documentation names. It never patches scripts or systems without permission, and records the reason for every change.

## Claude Code

Clone the repository into a folder named `power-ens` in the skills directory. For every project of the user:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.claude/skills/power-ens
```

For one project only, run from the project root:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git .claude/skills/power-ens
```

Start a new session. Claude Code loads the skill when a request matches its description. To load it explicitly, type `/power-ens`. Update it with `git -C ~/.claude/skills/power-ens pull`.

## OpenCode

OpenCode reads the Claude Code directories above, so one clone serves both tools. To keep it in the OpenCode directories, use the global or the project path:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.config/opencode/skills/power-ens
```

```bash
git clone https://github.com/arseniteio/skill-power-ens.git .opencode/skills/power-ens
```

The model loads the skill through its `skill` tool when a request matches the description. To load it explicitly, write `@power-ens` in the prompt. If `opencode.json` restricts skills with the `skill` permission, allow the resource `power-ens`.

## Use

Give the agent a root shell or sudo on the Wazuh server and on the machines to audit, and ask it to deploy Power ENS. It asks first for the deployment type, the machines, the ENS category and the restart window. Export the three passwords of the installation guide in the shell before you start; the agent never asks for them in the conversation.

## License

AGPL-3.0-only. See [LICENSE](LICENSE).
