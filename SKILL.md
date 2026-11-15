---
name: power-ens
description: Install, upgrade or uninstall OSCompliance ENS and the Power ENS plugin on Wazuh 4.14.7 to 4.14.9, audit Linux machines against the ENS, and bring them to compliance through configuration only (control parameters and per-control exclusions with a reason). Use when asked to deploy Power ENS, measure the ENS with Wazuh, or act on Power ENS verdicts.
---

# Power ENS

You install OSCompliance ENS and Power ENS on an existing Wazuh deployment: the rules, the engine and the plugin on the Wazuh server, and the agent on every Linux machine to audit. The engine writes a verdict per ENS control, requirement, evidence and machine every 15 minutes, and Power ENS shows them.

Your work has two parts. First install and verify. Then, if the person asks, read the verdicts and change the configuration so that it states what the organisation really does. You never change a system, a script or a binary to obtain a verdict.

The reference documentation is in two public repositories. Clone them on the machine you work from and read the files named below before you act.

```bash
git clone https://github.com/arseniteio/oscompliance-ens.git
git clone https://github.com/arseniteio/power-ens.git
```

| Need | File |
|---|---|
| Quick install with install.sh | `oscompliance-ens/README.md`, section Get started |
| Installation with packages, step by step | `oscompliance-ens/docs/en/install.md` |
| Installation with Docker | `oscompliance-ens/docs/en/install-docker.md` |
| ENS configuration and parameters from a file | `oscompliance-ens/docs/en/configuration.md` |
| Parameters of each control | `oscompliance-ens/docs/en/controls/` |
| Meaning of each status and cause | `oscompliance-ens/docs/en/verdicts.md` |
| Troubleshooting | `oscompliance-ens/docs/en/operations.md` |
| Plugin screens, access and removal | `power-ens/docs/en/usage.md`, `power-ens/docs/en/install.md` |

## Rules

1. Read each section of a guide before you run it, and run its commands as written. Change only the values the guide tells you to set.
2. Never invent an ENS parameter, the ENS category, the machines to audit or the reason for an exclusion. They belong to the organisation. If an answer is missing, ask. A control left without parameters shows as not evaluable by configuration, which is a correct result.
3. Never print, log, commit or ask for a password in the conversation. The person exports the variables of "Before you start" in the install guide; you check they are set without showing them and unset them when you finish.
4. After every step, run its check. If the result is not the expected one, read the troubleshooting section, fix only what it describes and repeat. If it still fails, stop and report the command, its output and what you tried.
5. Never patch the scripts or binaries of the repositories, and never change a system to make a verdict pass, unless the person asks for that specific change. If a script does not fit the deployment, stop and report what you would change and why.
6. Ask before anything that cannot be undone or that interrupts service: deleting indices, users or groups, and restarting a manager or dashboard in production outside the window the person gave.
7. Check the Wazuh and dashboard versions before anything else. Outside Wazuh 4.14.7 to 4.14.9, stop and ask; the person decides and you note it in the report.
8. Record every change you make with what changed, where, the value before and after, the reason and who approved it. The report includes that record.

## Before you install

Ask the person and write down the answers:

- The deployment type: all-in-one server installed from packages, the official single-node Docker compose file, or distributed.
- The Wazuh agent IDs to audit, and whether to audit the manager.
- The ENS category: BAJO, MEDIO or ALTO. A wrong category gives wrong verdicts without any error.
- The dashboard users who change the configuration, besides those with `all_access`.
- When the manager and the dashboard may restart.
- The control parameters they can give now. The rest can wait.

You need a root shell or sudo on the Wazuh server and on every machine to audit. Run every check of the Requirements and Before you start sections of `docs/en/install.md` on the server and on each machine. Do not continue with a check failing.

## Installation

### All-in-one server from packages

Use `install.sh`. It verifies every download against `SHA256SUMS`. Pin the release so that every machine gets the same version: replace `latest/download` in the URL of the README with `download/v<version>`, as below.

On the server, with the three password variables exported in the same shell:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens/releases/download/v1.1.0/install.sh \
  | sudo --preserve-env=API_ADMIN_PASSWORD,OSC_INDEXER_PASSWORD,OSC_WAZUH_API_PASSWORD bash -s -- server
```

The script installs the rules, the engine and its users and the plugin, and audits the manager. It asks for passwords only when the engine users do not exist yet. Without a terminal it cannot ask and stops with `no terminal to ask for passwords`; export the variables and run it again.

Then make the first save of the configuration. It can only be done in the dashboard: Power ENS, Configuración, Organización, fill in Cliente, set Nivel ENS and press Guardar configuración global. If you have a browser tool, do it with an `all_access` user; otherwise ask the person to do it and wait. The engine judges nothing and writes no group configuration until this save exists.

Add each machine to the `ens` group on the manager before you install its agent:

```bash
sudo /var/ossec/bin/agent_groups -a -i <id> -g ens -q
```

On each machine to audit:

```bash
curl -fsSL https://github.com/arseniteio/oscompliance-ens/releases/download/v1.1.0/install.sh | sudo bash -s -- agent
```

The agent role waits up to 15 minutes (`OSC_WAIT_SECONDS`) for the group configuration, restarts the Wazuh agent and runs the first full pass. If it times out, the machine is not in the `ens` group, or the first save is missing, or the engine has not synced the group yet. Check the three and run it again.

On a fleet, install machine by machine and write down the result of each one. Do not start the next batch while a machine of the previous batch has failed for a reason you do not understand.

### Fallback: verify the package by hand

When the person wants to check the files before running anything, or `curl | bash` is not allowed, download the package and verify it. The line of the package must end in `OK`. `install.sh` inside the package uses the files next to it. `uname -m` tells the architecture: `x86_64` is amd64 and `aarch64` is arm64.

```bash
v=1.1.0
url=https://github.com/arseniteio/oscompliance-ens/releases/download/v$v
curl -fLO "$url/oscompliance-ens-$v-linux-amd64.tar.gz" -O "$url/SHA256SUMS"
sha256sum -c --ignore-missing SHA256SUMS
tar xzf "oscompliance-ens-$v-linux-amd64.tar.gz"
sudo "./oscompliance-ens-$v-linux-amd64/install.sh" agent
```

On a server without access to GitHub, copy the plugin zip and `deploy-plugin.sh` of the same release into the package folder before `install.sh server`.

### Docker and distributed deployments

`install.sh server` only supports the all-in-one package installation. For Docker, follow `docs/en/install-docker.md` section by section. The agent role of `install.sh` works on the audited machines in every deployment type.

A distributed deployment, with a manager cluster, the indexer apart or several dashboards, is outside the guides. Tell the person and ask before going on. If they agree, follow `docs/en/install.md` and adapt only this:

- Rules: install them on every manager node and run `wazuh-analysisd -t` on each. Restart the nodes one at a time.
- Group and agents: create the `ens` group and add agents on the master. The cluster copies the group configuration to the workers.
- Engine: run it on a host that reaches the indexer on 9200 and the master's API on 55000, usually the indexer host. Point `engine-users.sh` there with `INDEXER_URL` and `API_URL`, and set `-indexer`, `-wazuh-api` and `-ca` in the service. Copy the API certificate, `/var/ossec/api/configuration/ssl/server.crt` on the master, to that host for `-wazuh-api-ca`, and keep `-wazuh-api-server-name` as the name in that certificate, `localhost` by default.
- Plugin: install the same zip on every dashboard, one at a time, and check each with `systemctl is-active wazuh-dashboard`.
- Access: run `create-admin-role.sh` on the indexer host, with `OS_URL` set to its address.
- Managers: every manager node is audited like any machine. Follow section 8 of the package guide on each one.

Write down every change from the guide in the report.

### Upgrade and uninstall

Upgrade with the same commands and a newer release: `server` on the server first, then `agent` on each machine. Remove everything with `uninstall` on each machine and on the server. The indices stay in the indexer; deleting them needs the person's approval.

## Pitfalls seen in real installations

- sudo drops the environment. Pass the password variables with `sudo --preserve-env=...`, and set `INDEXER_URL`, `API_URL` and `CERTS` the same way when they are not the defaults.
- The configuration document is `active` in `wazuh-ens-configuration`. `ACTIVE_CONFIGURATION` is its type, not its id.
- Do not build the initial configuration by hand through the plugin API. Use the first save in the dashboard.
- Behind a load balancer or an IP allowlist, the public dashboard URL can answer 403 to the server itself. Reach the dashboard from its own host or from an allowed address, never by opening the allowlist.
- Before editing `ossec.conf` or `agent.conf` by hand, keep a copy next to it with the date in the name, and say where in the report.
- Remove the plugin with `opensearch-dashboards-plugin remove powerEns`, never by deleting its folder.
- A compliance figure of a few percent right after the installation is normal: most controls are not evaluable until their parameters are declared.
- An application server, for example a mail or database server, needs no special installation. The agent collects the same facts on it. What differs is configuration: its service accounts, ports and paths are declared as parameters for that machine or its template.

## Verify

Run section 9 of `docs/en/install.md`. The installation is complete when every audited machine has 119001 alerts, there is one 119002 alert per machine and pass, the last engine pass shows `verdicts` above 0 and no `error`, and Power ENS shows a status with its reason for every evidence of a machine. Not evaluable by configuration is expected until the parameters are declared; not evaluable by collection is a fault to fix.

Without a browser, the plugin API confirms the dashboard side. Call it on the dashboard with the header `osd-xsrf: true` and a dashboard user that the person exports as `DASH_USER=user:password`:

```bash
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?include=summary"
```

It must answer 200 with `controlSummary` and `agentSummary`.

## Reaching compliance through configuration

Do this only when the person asks for it. Your aim is a configuration that states the truth about the organisation, not a green screen. A verdict that fails because the system really fails is a correct verdict, and you report it.

### What you may touch

- The Power ENS configuration: control parameters, templates and the system type of each machine, from the dashboard or with `set-parameters.py`.
- Per-control exclusions, `ens-excluir-<control>`, always with a reason.
- `/etc/oscompliance/agent.json` on a machine whose files are in non-standard paths. It holds paths only.
- The `agent.conf` of a Wazuh group outside the `oscompliance begin` and `oscompliance end` markers, for example to send an authentication log to Wazuh, with the person's approval. The engine owns everything between the markers; never edit it.

Everything else is outside your scope unless the person asks for that exact change: packages, services, accounts, sudoers, sshd, PAM, firewall, audit rules, the scripts and binaries of the repositories, Wazuh rules and `ossec.conf`.

### Never

- Never set or change a value only to make a verdict pass. Every value must describe what the organisation does or decided, and someone in the organisation must confirm it.
- Never widen a list, such as allowed accounts, ports or administrators, to include what the agent found without confirming that each item is legitimate.
- Never lower the ENS category, disable a control, remove a collector or remove a parameter to raise the score.
- Never write an attestation (P-ATT) without a real document: its reference, link, date and responsible person come from the organisation.
- Never exclude a machine from a control without a reason the person gave or approved.

### Procedure

1. Read the verdicts. Start with the summary, then each failing control on each machine:

   ```bash
   curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?include=summary"
   curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?controlId=op.acc.1&agentId=001"
   ```

   The same data is in the index `oscompliance-verdicts`: documents with `scope` (`evidence`, `requirement` or `control`), `control`, `agent.id`, `status`, `cause` and `reason`. Read `docs/en/verdicts.md` before you interpret them.

2. Classify each failing evidence by its status and cause, and read the control page in `docs/en/controls/` for the evidence and its parameters.

   - `NOT_EVALUABLE` by `CONFIGURATION`: a parameter is missing. Ask the organisation for its value, with the control page's description of what it means.
   - `NOT_EVALUABLE` by `COLLECTION`: a source did not arrive, such as the authentication log, auditd or an agent pass. Follow `docs/en/operations.md`. A fix on the machine needs the person's approval.
   - `NOT_EVALUABLE` by `EVIDENCE`: the data cannot decide the case. Report it for manual review.
   - `NON_COMPLIANT`: the reason names what failed. Decide with the person which case applies, in this order:
     a. The system is wrong. Report it with the reason and what would fix it. Fixing it is outside your scope unless the person asks.
     b. The parameter does not describe the organisation, for example a service account the organisation approved is missing from the list of allowed accounts. Propose the corrected value and apply it once confirmed.
     c. The control does not apply to that machine, for example a measure that only makes sense on a kind of machine this one is not. Propose an exclusion with the reason and apply it once approved.
   - A requirement `NON_COMPLIANT` and not evidenced, with no technical evidence, needs an attestation row in P-ATT with data from the organisation.

3. Choose where the value lives. Each parameter has a class, listed on its control page:

   - `organisation`: only the common value of the control counts; per-machine values are ignored.
   - `system` and `platform`: the machine's own value wins over its template, and the template wins over the common value.

   Put a value at the narrowest level that matches the fact: the machine in Agentes when it is true of one machine, a template in Plantillas when it is true of a kind of machine, the common value when it is true of the whole organisation. A value for one machine never goes to the template or the common value.

4. Apply parameters. In the dashboard, use Configuración, Plantillas or Agentes. From a file, use `power-ens/config/set-parameters.py` as described in "Parameters from a file" of `docs/en/configuration.md`: dry run first, `--apply` only when the plan shows no `ERROR` and only the changes you intend. Never send empty values or `null`: they remove the current value. Keep the file out of any repository and delete it when you finish. Each save keeps the previous version in the configuration history.

5. Apply exclusions. The option Grupos por control in Organización must be on; the engine then creates one Wazuh group per control. The manager (agent 000) belongs to no group and cannot be excluded.

   With a browser, open Detalle del control on the machine, press Excluir de este control, write the reason and confirm. Power ENS moves the machine to the group and records agent, control, action, reason, user and date in the index `wazuh-ens-exclusions`. Volver a incluir reverses it, also with a reason.

   Without a browser, do the same two steps, in this order, with a dashboard user that has `power_ens_admin` or `all_access`:

   ```bash
   sudo /var/ossec/bin/agent_groups -a -i 001 -g ens-excluir-op.acc.1 -q
   curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' -H 'Content-Type: application/json' \
     -X POST "https://<dashboard>/api/powerEns/exclusions" \
     -d '{"agentId":"001","controlId":"op.acc.1","action":"EXCLUDED","reason":"<reason given by the organisation>"}'
   ```

   If the record fails, undo the group change with `agent_groups -r -i 001 -g ens-excluir-op.acc.1 -q`, so that no exclusion exists without its reason. To include the machine again, remove it from the group and record `"action":"INCLUDED"` with a reason. The reason holds up to 1000 characters; write who decided and why, not only that the control does not apply.

6. Check the effect. Wait for the next engine pass, about 15 minutes, and read the verdicts again. Each change must have the effect you expected and nothing else may have changed. If a change had another effect, report it and ask before reverting.

7. Record. For every change write the control, the parameter or exclusion, the level (common, template or machine), the value before and after, the reason, who approved it and the date.

## Report

When you finish, or when you stop, report in this order:

1. What is installed, with versions, and on which servers.
2. The machines in the `ens` group and whether each one sends facts.
3. The result of every verification check.
4. The configuration changes and exclusions you made, from the record of rule 8.
5. The verdicts that still fail, grouped by cause: system faults to fix, parameters the organisation must still declare, collection faults, cases for manual review.
6. What you could not do, and why.
