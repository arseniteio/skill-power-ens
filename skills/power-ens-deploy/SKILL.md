---
name: power-ens-deploy
description: Install, upgrade or uninstall OSCompliance ENS and the Power ENS plugin on Wazuh 4.14.7 to 4.14.9, all-in-one, Docker or distributed, and add Linux machines to the audit. Use when asked to deploy, install, upgrade or remove Power ENS or OSCompliance ENS, or to measure the ENS with Wazuh.
---

# Deploy Power ENS

You install OSCompliance ENS and Power ENS on an existing Wazuh deployment. The installer does the work: `install.sh server` detects what each Wazuh node is and installs only its part, and `install.sh agent` adds a Linux machine to the audit. You orchestrate: you ask, find the role of each host, plan the order, run one command per host and verify.

The engine runs on the master manager and writes a verdict per ENS control and machine every 15 minutes. Power ENS shows them in the Wazuh dashboard.

You work in five steps: ask, discover, plan, install, verify. You never skip the plan.

## Rules

1. Ask before you assume. The ENS category, the machines to audit, the control parameters and the restart window belong to the organisation. Never invent them.
2. Never type, print, log, commit or ask for a password in the conversation, and never put one on a command line. The operator types it at the installer prompt in an interactive terminal, or exports it in the session where the installer runs. You check that a variable is set without showing it. Never copy SSH keys or client data anywhere.
3. Run one installer command per host, as written in this skill. Never replay the installer's steps by hand.
4. After every host, run its check. If it fails, read the troubleshooting section of `docs/en/operations.md` in the oscompliance-ens-wazuh repository, fix only what it describes and run the same command again. If it still fails, stop and report the command, its output and what you tried.
5. Never patch the scripts or binaries of the repositories. If something does not fit, stop and explain what you would change and why.
6. Ask before anything that cannot be undone or that interrupts service: deleting indices, users or groups, and running the installer on a manager or dashboard outside the restart window. The installer restarts `wazuh-manager` on managers and `wazuh-dashboard` on dashboards.
7. Keep a record of every host, command, check and result. It becomes the report.

## 1. Ask

Ask these questions in one message and wait for the answers:

1. What to do: a new installation, add machines to an existing one, upgrade, or uninstall.
2. Where Wazuh runs and how you reach each host: this shell, SSH host aliases, a bastion. Whether sudo asks for a password.
3. The machines to audit, by host name, and how to reach them: SSH, or a fleet tool such as Ansible or Satellite that the operator runs.
4. The ENS category, BAJO, MEDIO or ALTO, and the client name shown in Power ENS. A wrong category gives wrong verdicts without any error.
5. The dashboard users who will change the ENS configuration, besides those with `all_access`.
6. When the managers and dashboards may restart.
7. The release to install. Default: the latest release of `arseniteio/oscompliance-ens-wazuh`, pinned for the whole run.

If sudo needs a password on a host, do not type it. Ask the operator to run the command on that host.

## 2. Discover

Find the role of every Wazuh host over SSH, as described in [distributed.md](distributed.md). Read it even for one host: it lists what to look for and what the installer does on each role.

Decide the deployment type:

- Package install, one host or several: the installer handles it. One host with manager, indexer and dashboard is a single master node.
- Docker: the official single-node Wazuh compose file. The installer does not detect nodes inside containers. Follow `docs/en/install-docker.md` of oscompliance-ens-wazuh instead of step 4.

Stop and ask if the Wazuh version is outside 4.14.7 to 4.14.9, or if the dashboard is not 2.19.5 on Wazuh 4.14.7 or 2.19.6 on 4.14.8 and 4.14.9.

Resolve the release once and pin it for every host, so that all of them get the same version:

```bash
curl -fsSLI -o /dev/null -w '%{url_effective}\n' https://github.com/arseniteio/oscompliance-ens-wazuh/releases/latest
```

On each machine to audit, check that `wazuh-agent`, `auditd` and `cron` (`crond` on RHEL) are active and that the authentication log reaches Wazuh, as in "Before you start" of `docs/en/install.md`. The installer only warns when `auditd` or `cron` is missing.

## 3. Plan

Write the plan and show it to the operator: each host, its role, the command you will run on it, the order, and what restarts. List every failed check from step 2 with its fix. Wait for the operator's go. Do not start with a check failing.

The order is fixed: the master first, then the workers one at a time, then the dashboards one at a time, then the audited machines. Indexer-only nodes get nothing.

## 4. Install

Every command pins the release with `OSC_RELEASE`. The examples use 1.2.0.

### Master manager

The installer asks for up to four passwords on the master. The operator finds the existing ones in `wazuh-passwords.txt` inside `wazuh-install-files.tar`, and chooses the two new ones:

| Prompt | Variable |
| --- | --- |
| Wazuh API user `wazuh-wui` | `API_ADMIN_PASSWORD` |
| New password of the engine indexer user | `OSC_INDEXER_PASSWORD` |
| New password of the engine Wazuh API user: 8 to 64 characters with upper and lower case, a digit and a symbol, and no `$`, `&`, quotes, backslashes or spaces | `OSC_WAZUH_API_PASSWORD` |
| Indexer user `admin`, only when the master has no indexer admin certificate | `INDEXER_ADMIN_PASSWORD` |

A non-interactive SSH session has no terminal for the prompts and does not carry your environment, so you do not run this command on a remote master. Give the operator the command and ask them to run it in their own terminal on the master, for example after `ssh -t wazuh-mgr-01`:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens-wazuh/releases/download/v1.2.0/install.sh \
  | sudo OSC_RELEASE=1.2.0 bash -s -- server
```

If your shell runs on the master itself and the operator exported the variables there, run it yourself and keep them through sudo:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens-wazuh/releases/download/v1.2.0/install.sh \
  | sudo --preserve-env=API_ADMIN_PASSWORD,OSC_INDEXER_PASSWORD,OSC_WAZUH_API_PASSWORD,INDEXER_ADMIN_PASSWORD \
    OSC_RELEASE=1.2.0 bash -s -- server
```

Without a terminal and without the variables, the installer stops with `no terminal to ask for it`.

The installer reads the indexer URLs from `/etc/filebeat/filebeat.yml` on the master. If that fails, the operator sets `OSC_INDEXER_URLS` and `OSC_INDEXER_CA` and runs it again.

Ask the operator for the installer output and check that it ends with `Done on this node`.

### Workers and dashboards

These need no password. Run the same command on each one over SSH, in the order of the plan:

```bash
ssh wazuh-worker-01 'curl -fsSL https://github.com/arseniteio/oscompliance-ens-wazuh/releases/download/v1.2.0/install.sh | sudo OSC_RELEASE=1.2.0 bash -s -- server'
```

A worker waits up to 15 minutes for the rules of the release, which the cluster syncs from the master, and then restarts its manager. If they do not arrive it stops with `the rules of release 1.2.0 have not arrived from the master`: check the master and `cluster_control -l`, and run it again.

Do not run it on indexer-only nodes. It only answers `indexer nodes need nothing`.

For a layout the detection does not fit, `--only` replaces it with a comma-separated list of `rules`, `audit`, `engine`, `plugin` and `worker`. Use it only with the operator's approval, and record why.

### Without curl | bash

If the operator does not allow `curl | bash`, or a host has no Internet access, use the release tarball: download `oscompliance-ens-wazuh-1.2.0-linux-<arch>.tar.gz` and `SHA256SUMS`, run `sha256sum -c --ignore-missing SHA256SUMS`, extract it and run `sudo ./install.sh server` from the extracted folder. On a dashboard, put the plugin zip and `deploy-plugin.sh` of the power-ens-wazuh release in the same folder; the installer uses them instead of downloading.

### Configuration

The engine judges with level MEDIO and scope group `ens` until a configuration is saved in Power ENS. If the category from step 1 is not MEDIO, or to set the client name, save it in Power ENS, Configuración, Organización, with an `all_access` user: with a browser tool, or by asking the operator. The power-ens-compliance skill covers later changes.

To allow the configuration users of step 1, run `create-admin-role.sh --map <user>` from the power-ens-wazuh release on a host with the indexer admin certificates.

### Audited machines

Run on each machine:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens-wazuh/releases/download/v1.2.0/install.sh \
  | sudo OSC_RELEASE=1.2.0 bash -s -- agent --group ens
```

The engine adds the machine to the `ens` group within 15 minutes, and the group configuration follows. You do not run `agent_groups` on the manager. With a fleet tool, the operator runs the same command on every machine.

Install one machine first and verify it. Then go on in batches, and stop a batch when a machine fails for a reason you do not understand.

An application server, such as a mail or database server, needs nothing special. What differs is configuration, handled by the power-ens-compliance skill.

### Upgrade

Run the same command on each node with the new release, in the same order: the master first. Then the audited machines.

### Uninstall

Run `install.sh uninstall` on each audited machine, then on each dashboard and worker, then on the master. On the master it asks again for the passwords that remove the engine users, so the operator runs it there. The indices stay; deleting them needs the operator's approval.

## 5. Verify

The installation is complete when:

- on the master, `systemctl status oscompliance-engine` is active and `journalctl -u oscompliance-engine` shows `pass` lines with `verdicts` above 0 and no `error`;
- the index `oscompliance-verdicts` holds documents;
- every audited machine is in the group: `sudo /var/ossec/bin/agent_groups -l -g ens` on the master;
- every audited machine has `rule.id:119001` alerts, and one `rule.id:119002` alert per pass;
- each dashboard shows Power ENS with a status and its reason for every evidence of a machine.

Without a browser, check the dashboard side with a dashboard user the operator exports as `DASH_USER=user:password`. It must answer 200 with `controlSummary` and `agentSummary`:

```bash
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?include=summary"
```

Low compliance right after the installation is expected: controls stay not evaluable by configuration until their parameters are declared. Not evaluable by collection is a fault to fix.

## 6. Report

Report in this order:

1. What is installed, with versions, on which hosts and roles.
2. Every deviation from this skill, such as `--only`, and why.
3. The machines in the `ens` group and whether each one sends facts.
4. The result of every verification check.
5. What you could not do, and why.

Then offer the power-ens-compliance skill to adjust the level, the client name and the scope, and to declare the parameters and act on the verdicts.
