---
name: power-ens
description: Install, upgrade or uninstall OSCompliance ENS (agent, engine, rules) and the Power ENS plugin on Wazuh 4.14.7 to 4.14.9, from packages or Docker, following the documentation of their repositories. Use when asked to deploy Power ENS or to measure the ENS with Wazuh.
---

# Power ENS

You install OSCompliance ENS and Power ENS on an existing Wazuh deployment: rules and the engine on the Wazuh server, the plugin on the dashboard and the agent on every Linux machine to audit. The result is a verdict per ENS control and machine in Power ENS, refreshed every 15 minutes.

The procedure is in the documentation of two private repositories. Clone them first; everything below refers to their files.

```bash
git clone https://github.com/arseniteio/oscompliance-ens.git
git clone https://github.com/arseniteio/power-ens.git
```

| Need | File |
|---|---|
| Installation with packages | `oscompliance-ens/docs/en/install.md` |
| Installation with Docker | `oscompliance-ens/docs/en/install-docker.md` |
| ENS configuration, also from a file | `oscompliance-ens/docs/en/configuration.md` |
| Control parameters | `oscompliance-ens/docs/en/controls/` |
| Troubleshooting | `oscompliance-ens/docs/en/operations.md` |
| Plugin build, access and removal | `power-ens/docs/en/install.md` |

If cloning fails, stop: the person must give you read access to the arseniteio repositories.

## Rules

1. Read each section of the guide before you run it, and run its commands as written. Change only the values the guide tells you to set.
2. Never invent an ENS parameter, the ENS category or the machines to audit. They belong to the organisation. If an answer is missing, ask. A control without parameters is left not evaluable by configuration and reported, which is a correct result.
3. Never print, log, commit or ask for a password in the conversation. The person exports the three variables of "Before you start" in the install guide; you check they are set without showing them and unset them when you finish.
4. After every section, run its check. If the result is not the expected one, read the troubleshooting section, fix only what it describes and repeat. If it still fails, stop and report the command, its output and what you tried.
5. Do not change anything the guides do not change: the Wazuh compose file, the `ossec.conf` of the agents, other rules, groups or users.
6. Ask before anything that cannot be undone or that interrupts service: deleting indices, users or groups, and restarting a manager or dashboard in production outside the window the person gave.
7. Never patch the scripts of the repositories. If one does not fit the deployment, stop and report what you would need to change and why.
8. Check the Wazuh and dashboard versions before anything else. If the Wazuh version is outside the range the guide supports, stop and ask: the person decides whether to go on, and you note it in the report.

## Before you install

Ask the person and write down the answers:

- Packages with everything on one server, or the official single-node Docker compose file.
- The Wazuh agent IDs to audit, and for packages whether to audit the manager too.
- The ENS category: BAJO, MEDIO or ALTO. A wrong category gives wrong verdicts without any error.
- The dashboard users who change the configuration, besides those with `all_access`.
- When the manager and the dashboard may restart.
- The control parameters they can give now. The rest can wait.

You need a root shell on the Wazuh server or Docker host and on every machine to audit, and a build machine with Go and Docker. Then run every check of the Requirements and Before you start sections of the guide. Do not continue with a check failing.

## Installation

Follow the guide section by section. These are the points where agents usually go wrong:

- Section 1: build the plugin for the dashboard version you checked, 2.19.5 or 2.19.6. A zip only works on its version. The first build takes about 20 minutes; let it finish.
- Rules: the check of the 119000 to 119004 range must print nothing before you install them.
- Engine users: with the three variables exported, the script asks nothing. It must end with `credentials written to /etc/oscompliance/engine.env`.
- ENS configuration: the first save, Guardar configuración global in Power ENS, can only be done from the dashboard. If you have a browser tool, do it with an `all_access` user; otherwise ask the person to do it and wait. After it, set the parameters the person gave with `set-parameters.py`, as in "Parameters from a file" of the configuration page: dry run first, `--apply` only when the plan has no `ERROR`. Never send empty values: they remove the current ones.
- Engine: do not start it before the first save. Its log must show `group configuration` with `applied` true.
- Agents: keep the order of the section. The Wazuh agent must have the facts localfile and be restarted before the first `oscompliance-agent -full`, or Wazuh skips the first facts.
- Docker: the guide adds `docker-compose.override.yml` next to the compose file. If one already exists, merge it by hand and tell the person.

## Pitfalls seen in real installations

- Without root, run the scripts with `sudo`. sudo drops the environment, so the password variables do not reach the script: pass them with `sudo --preserve-env=API_ADMIN_PASSWORD,OSC_INDEXER_PASSWORD,OSC_WAZUH_API_PASSWORD` and set `INDEXER_URL`, `API_URL` and `CERTS` the same way when they are not the defaults.
- The configuration document is `active` in `wazuh-ens-configuration`. Older notes call it `ACTIVE_CONFIGURATION`, which is its type, not its id.
- Do not build the initial ENS configuration by hand through the plugin API. It needs the full catalog of controls and any mistake is saved as policy. Use the first save in the dashboard.
- Behind a load balancer or an IP allowlist, the public dashboard URL can answer 403 to the server itself. Reach the dashboard from its own host or from an allowed address, never by opening the allowlist.
- Before editing `ossec.conf` on a manager, keep a copy next to it with the date in the name, and say where in the report.
- To remove the plugin, use `opensearch-dashboards-plugin remove powerEns` as the guide says. Deleting its folder skips the cleanup the dashboard does itself.
- If Go or Docker are missing on the build machine, ask before installing them. Use the Go version that `go.mod` asks for or a later one.
- A compliance figure of a few percent right after the installation is normal: most controls are not evaluable until their parameters are declared.

## Deployments the guides do not cover

The guides cover everything on one server and the official single-node Docker compose file. In a distributed deployment, with a manager cluster, the indexer apart or several dashboards, tell the person that it is outside the guides and ask before going on. If they agree, keep to the guide and adapt only this:

- Rules: install them on every manager node, master and workers, and run `wazuh-analysisd -t` on each one. Restart the nodes one at a time.
- Group and agents: create the `ens` group and add agents on the master. The cluster copies the group configuration to the workers.
- Engine: run it on a host that reaches the indexer on 9200 and the master's API on 55000, usually the indexer host. Point `engine-users.sh` there with `INDEXER_URL` and `API_URL`, and in the service set `-indexer`, `-wazuh-api` and `-ca`. Copy the API certificate, `/var/ossec/api/configuration/ssl/server.crt` on the master, to that host for `-wazuh-api-ca`, and keep `-wazuh-api-server-name` as the name in that certificate, `localhost` by default.
- Plugin: install the same zip on every dashboard, one at a time, so the load balancer always has one serving. Check each with `systemctl is-active wazuh-dashboard`.
- Access: run `create-admin-role.sh` on the indexer host, with `OS_URL` set to its address.
- Auditing the managers: every manager node is a machine like any other. Follow section 8 of the package guide on each one.

Write down every change from the guide in the report, so the person can tell what is standard and what is not.

## Verify

Run the verification section of the guide. The installation is complete when every audited machine has 119001 alerts, there is one 119002 alert per machine and run, the last engine pass shows `verdicts` above 0 and no `error`, and Power ENS shows a status with its reason for every evidence of a machine. Partial results caused by parameters that are not declared yet are expected; not evaluable by collection is a fault to fix.

Without a browser, the plugin API confirms the dashboard side: `GET /api/powerEns/evaluation?include=summary` on the dashboard, with a dashboard user and the header `osd-xsrf: true`, must answer 200 with `controlSummary` and `agentSummary`.

## Report

When you finish, or when you stop, report in this order:

1. What is installed, with versions, and on which servers.
2. The machines in the `ens` group and whether each one sends facts.
3. The result of every verification check.
4. The controls that stay not evaluable by configuration, with the parameters they need.
5. What you could not do, and why.
