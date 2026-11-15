---
name: power-ens-deploy
description: Install, upgrade or uninstall OSCompliance ENS and the Power ENS plugin on Wazuh 4.14.7 to 4.14.9, all-in-one, Docker or distributed, and add Linux machines to the audit. Use when asked to deploy, install, upgrade or remove Power ENS or OSCompliance ENS, or to measure the ENS with Wazuh.
---

# Deploy Power ENS

You install OSCompliance ENS and Power ENS on an existing Wazuh deployment: the rules on every manager, the engine on one host, the plugin on every dashboard, and the agent on every Linux machine to audit. The engine writes a verdict per ENS control and machine every 15 minutes, and Power ENS shows them in the Wazuh dashboard.

You work in four steps: ask, discover, plan, install. You never skip the plan.

## Rules

1. Ask before you assume. The ENS category, the machines to audit, the control parameters and the restart window belong to the organisation. Never invent them.
2. Never print, log, commit or ask for a password in the conversation. The operator exports them in the shell you use, and you check that they are set without showing them. Never copy SSH keys or client data anywhere.
3. Run the commands of the guides as written. Change only the values they tell you to set.
4. After every step, run its check. If it fails, read the troubleshooting section of `docs/en/operations.md`, fix only what it describes and repeat. If it still fails, stop and report the command, its output and what you tried.
5. Never patch the scripts or binaries of the repositories, and never change anything the guides do not change. If something does not fit, stop and explain what you would change and why.
6. Ask before anything that cannot be undone or that interrupts service: deleting indices, users or groups, and restarting a manager or dashboard outside the window the operator gave.
7. Before editing `ossec.conf`, `agent.conf` or a systemd unit, keep a copy next to it with the date in the name.
8. Keep a record of every host, command, check and result. It becomes the report.

## 1. Ask

Ask these questions in one message and wait for the answers:

1. What to do: a new installation, add machines to an existing one, upgrade, or uninstall.
2. Where Wazuh runs and how you reach each host: this shell, SSH host aliases, a bastion. Whether sudo asks for a password.
3. The machines to audit, by Wazuh agent ID or host name, and whether to audit the Wazuh managers too.
4. The ENS category, BAJO, MEDIO or ALTO, and the client name shown in Power ENS. A wrong category gives wrong verdicts without any error.
5. The dashboard users who will change the ENS configuration, besides those with `all_access`.
6. When the managers and dashboards may restart.
7. The release to install. Default: the latest release of `arseniteio/oscompliance-ens`, pinned for the whole run.

If sudo needs a password on a host, do not type it. Ask the operator to run the step, or to open a session where sudo does not ask.

## 2. Discover

Clone the documentation where you work, and keep both clones for the whole run:

```bash
git clone https://github.com/arseniteio/oscompliance-ens.git
git clone https://github.com/arseniteio/power-ens.git
```

On each Wazuh host, find its roles and versions:

```bash
sudo /var/ossec/bin/wazuh-control info
sudo /var/ossec/bin/cluster_control -l
systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
grep -m1 '"version"' /usr/share/wazuh-dashboard/package.json
docker ps --format '{{.Image}} {{.Names}}' | grep -i wazuh
```

`WAZUH_TYPE="server"` is a manager. `cluster_control -l` lists the cluster nodes, or fails when there is no cluster. Commands for roles the host does not have fail; that is the answer.

Decide the deployment type:

- All-in-one: manager, indexer and dashboard on one host, installed from packages.
- Docker: the official single-node Wazuh compose file.
- Distributed: any other layout, such as a manager cluster, the indexer apart or several dashboards.

Stop and ask if the Wazuh version is outside 4.14.7 to 4.14.9, or if the dashboard is not 2.19.5 on Wazuh 4.14.7 or 2.19.6 on 4.14.8 and 4.14.9.

On each machine to audit, run the checks of "Before you start" in `oscompliance-ens/docs/en/install.md`: `wazuh-agent`, `auditd` and `cron` (`crond` on RHEL) active, the architecture, and the authentication log reaching Wazuh.

## 3. Plan

Write the plan and show it to the operator: each host, its roles, what you will install on it, in which order, and what restarts. List every failed check from step 2 with its fix. Wait for the operator's go. Do not start with a check failing.

## 4. Install

Pin the release in every URL: `releases/download/v<version>/`, never `latest`, so every host gets the same version. The examples use 1.1.0.

### All-in-one

On the server, with `API_ADMIN_PASSWORD`, `OSC_INDEXER_PASSWORD` and `OSC_WAZUH_API_PASSWORD` exported (see "Before you start" in the install guide):

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens/releases/download/v1.1.0/install.sh \
  | sudo --preserve-env=API_ADMIN_PASSWORD,OSC_INDEXER_PASSWORD,OSC_WAZUH_API_PASSWORD bash -s -- server
```

It installs the rules, the engine and its users and the plugin, and audits the manager. Without a terminal and without the variables it stops with `no terminal to ask for passwords`.

If the operator does not allow `curl | bash`, use the verified package instead, as in "Get started" of `oscompliance-ens/README.md`: download the tarball and `SHA256SUMS`, run `sha256sum -c --ignore-missing SHA256SUMS`, and run `install.sh` from the extracted folder.

### Docker

Follow `oscompliance-ens/docs/en/install-docker.md` section by section. If a `docker-compose.override.yml` already exists, merge it by hand and tell the operator.

### Distributed

Follow [distributed.md](distributed.md). It maps each section of the package guide to the host where it runs.

### First save

When the plugin is installed, the ENS configuration must be created once from the dashboard: Power ENS, Configuración, Organización, fill in Cliente, set Nivel ENS and press Guardar configuración global. With a browser tool, do it with an `all_access` user; otherwise ask the operator and wait. The engine judges nothing and writes no group configuration until then.

To allow the configuration users of step 1, run `create-admin-role.sh --map <user>` from the power-ens release on the indexer host.

### Audited machines

Add each machine to the `ens` group on the manager, or on the master in a cluster:

```bash
sudo /var/ossec/bin/agent_groups -a -i <id> -g ens -q
```

Then, on each machine:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens/releases/download/v1.1.0/install.sh | sudo bash -s -- agent
```

It waits up to 15 minutes (`OSC_WAIT_SECONDS`) for the group configuration, restarts the Wazuh agent and runs the first full pass. If it times out, check that the machine is in the `ens` group, that the first save exists and that the engine log shows `group configuration` with `applied` true, then run it again.

Install one machine first and verify it. Then go on in batches, and stop a batch when a machine fails for a reason you do not understand.

An application server, such as a mail or database server, needs nothing special: the agent collects the same facts on any Linux machine. What differs is configuration, handled by the power-ens-compliance skill.

### Upgrade and uninstall

Upgrade with the same commands and the newer release: the server side first, then each machine. Uninstall with `install.sh uninstall` on each machine and then on the server, or with the Uninstall section of the install guide on Docker and distributed deployments. The indices stay; deleting them needs the operator's approval.

## 5. Verify

Run section 9 of `oscompliance-ens/docs/en/install.md`. The installation is complete when:

- every audited machine has `rule.id:119001` alerts, and one `rule.id:119002` alert per pass;
- the last engine `pass` log line shows `verdicts` above 0 and no `error`;
- Power ENS shows a status with its reason for every evidence of a machine.

Without a browser, check the dashboard side with a dashboard user the operator exports as `DASH_USER=user:password`. It must answer 200 with `controlSummary` and `agentSummary`:

```bash
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?include=summary"
```

Low compliance right after the installation is expected: controls stay not evaluable by configuration until their parameters are declared. Not evaluable by collection is a fault to fix.

## 6. Report

Report in this order:

1. What is installed, with versions, on which hosts.
2. Every deviation from the guides, and why.
3. The machines in the `ens` group and whether each one sends facts.
4. The result of every verification check.
5. What you could not do, and why.

Then offer the power-ens-compliance skill to declare the parameters and act on the verdicts.
