# OSCompliance ENS and Power ENS skills

Agent skills that deploy [OSCompliance ENS](https://github.com/arseniteio/oscompliance-ens) and [Power ENS](https://github.com/arseniteio/power-ens) on Wazuh and bring Linux machines to ENS compliance through configuration only. They work with Claude Code and OpenCode.

## How it works

Ask your agent to deploy Power ENS. It asks what it needs first: the task, the Wazuh hosts and how to reach them, the machines to audit, the ENS category and the restart window. It then detects the Wazuh layout and versions on each host, shows a plan and waits for your go. It installs from the published release, checks every step and ends with a report.

Then ask it to raise compliance. It reads the verdicts, separates real system faults from missing or wrong configuration, and proposes parameter values or exclusions with a reason. It applies only what you confirm, and records why for every change. It never patches scripts or systems without your permission, and never changes a value only to turn a verdict green.

Passwords stay in your shell. Export them before you start; the agent never asks for them in the conversation.

## Installation

### Claude Code

```
/plugin install power-ens --marketplace arseniteio/skill-power-ens
```

On Claude Code before 2.1.275, add the marketplace first:

```
/plugin marketplace add arseniteio/skill-power-ens
/plugin install power-ens@arseniteio
```

Update with `/plugin marketplace update arseniteio`.

### OpenCode

Clone the repository into the OpenCode skills directory. OpenCode finds both skills inside it.

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.config/opencode/skills/power-ens
```

For one project only, clone it into `.opencode/skills/power-ens` at the project root. Update with `git -C ~/.config/opencode/skills/power-ens pull`. If `opencode.json` restricts skills with the `skill` permission, allow `power-ens-*`.

### Check

Start a new session and ask: "What Power ENS skills do you have?" The agent lists power-ens-deploy and power-ens-compliance.

## Skills

- power-ens-deploy installs, upgrades or uninstalls OSCompliance ENS and Power ENS on Wazuh 4.14.7 to 4.14.9, all-in-one, on Docker or distributed, and adds machines to the audit.
- power-ens-compliance reads the verdicts and declares control parameters and per-control exclusions with a reason.

The skills load on their own when a request matches. To load one explicitly, use `/power-ens:power-ens-deploy` in Claude Code or `@power-ens-deploy` in OpenCode.

## License

AGPL-3.0-only. See [LICENSE](LICENSE).
