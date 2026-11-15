---
name: power-ens-compliance
description: Read Power ENS verdicts and bring Linux machines to ENS compliance through configuration only, by declaring control parameters and excluding machines from controls with a recorded reason. Use when asked to configure Power ENS, declare ENS parameters, explain why a control fails, exclude a machine from a control, or raise ENS compliance.
---

# Reach ENS compliance through configuration

Power ENS shows one verdict per ENS control, requirement, evidence and machine. You read the verdicts and change the configuration so that it states what the organisation really does. Your aim is a true configuration, not a green screen. A verdict that fails because the system fails is correct, and you report it.

This skill assumes Power ENS is installed. If it is not, use the power-ens-deploy skill first.

## Rules

1. Never set or change a value only to make a verdict pass. Every value describes what the organisation does or decided, and someone in the organisation confirms it.
2. Never widen a list, such as allowed accounts, ports or administrators, to include what the agent found without confirming that each item is legitimate.
3. Never lower the ENS category, disable a control, remove a collector or remove a parameter to raise the score.
4. Never write an attestation (P-ATT) without a real document. Its reference, link, date and responsible person come from the organisation.
5. Never exclude a machine from a control without a reason the operator gave or approved.
6. Never print or ask for a password in the conversation. The operator exports a dashboard user as `DASH_USER=user:password` in the shell you use.
7. Record every change: control, parameter or exclusion, level, value before and after, reason, who approved it and the date.

## What you may change

- The Power ENS configuration: control parameters, templates and the system type of each machine, in the dashboard or with `set-parameters.py`.
- Per-control exclusions, `ens-excluir-<control>`, always with a reason.
- `/etc/oscompliance/agent.json` on a machine whose files are in non-standard paths. It holds paths only.
- The `agent.conf` of a Wazuh group outside the `oscompliance begin` and `oscompliance end` markers, for example to send an authentication log to Wazuh, with the operator's approval and a dated copy first. The engine owns everything between the markers; never edit it.

Everything else is outside your scope unless the operator asks for that exact change: packages, services, accounts, sudoers, sshd, PAM, firewall, audit rules, the scripts and binaries of the repositories, Wazuh rules and `ossec.conf`.

## Before you start

Clone the documentation and read `docs/en/verdicts.md` and `docs/en/configuration.md`:

```bash
git clone https://github.com/arseniteio/oscompliance-ens.git
git clone https://github.com/arseniteio/power-ens.git
```

Ask the operator, in one message: the dashboard URL, which controls or machines to work on (all by default), and who in the organisation confirms values and exclusions.

## 1. Read the verdicts

Start with the summary, then each failing control on each machine:

```bash
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?include=summary"
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' "https://<dashboard>/api/powerEns/evaluation?controlId=op.acc.1&agentId=001"
```

The same data is in the index `oscompliance-verdicts`: documents with `scope` (`evidence`, `requirement` or `control`), `control`, `agent.id`, `status`, `cause` and `reason`.

## 2. Classify

For each failing evidence, read its control page in `oscompliance-ens/docs/en/controls/` and classify it:

- `NOT_EVALUABLE` by `CONFIGURATION`: a parameter is missing. Ask the organisation for its value, quoting what the control page says it means.
- `NOT_EVALUABLE` by `COLLECTION`: a source did not arrive, such as the authentication log, auditd or an agent pass. Follow `docs/en/operations.md`. A fix on the machine needs the operator's approval.
- `NOT_EVALUABLE` by `EVIDENCE`: the data cannot decide the case. Report it for manual review.
- `NON_COMPLIANT`: the reason names what failed. Decide with the operator which case applies, in this order:
  1. The system is wrong. Report it with what would fix it. Fixing it is outside your scope unless the operator asks.
  2. The parameter does not describe the organisation, for example an approved service account missing from the allowed accounts. Propose the corrected value and apply it once confirmed.
  3. The control does not apply to that machine. Propose an exclusion with its reason and apply it once approved.
- A requirement `NON_COMPLIANT` and not evidenced, with no technical evidence, needs an attestation row in P-ATT (`requisito | referencia | enlace | fecha | responsable`) with data from the organisation.

Show the operator the list of proposals, grouped by case, before you change anything.

## 3. Choose the level

Each parameter has a class, listed on its control page:

- `organisation`: only the common value of the control counts. Per-machine values are ignored.
- `system` and `platform`: the machine's own value wins over its template, and the template wins over the common value.

Put each value at the narrowest level that matches the fact: the machine in Agentes when it is true of one machine, a template in Plantillas when it is true of a kind of machine, the common value when it is true of the whole organisation. A value for one machine never goes to a template or the common value.

## 4. Apply parameters

In the dashboard, use Configuración, Plantillas or Agentes. From a file, use `power-ens/config/set-parameters.py` as described in "Parameters from a file" of `docs/en/configuration.md`:

```bash
export OS_CERT=/etc/wazuh-indexer/certs/admin.pem OS_KEY=/etc/wazuh-indexer/certs/admin-key.pem
python3 set-parameters.py /root/parameters.json
python3 set-parameters.py /root/parameters.json --apply
```

Run `--apply` only when the dry run shows no `ERROR` and only the changes you intend. Never send empty values or `null`: they remove the current value. Keep the file out of any repository and delete it when you finish. Each save keeps the previous version in the configuration history.

## 5. Apply exclusions

The option Grupos por control in Organización must be on; the engine then creates one Wazuh group per control. The manager (agent 000) belongs to no group and cannot be excluded.

With a browser, open Detalle del control on the machine, press Excluir de este control, write the reason and confirm. Power ENS moves the machine to the group and records agent, control, action, reason, user and date in the index `wazuh-ens-exclusions`. Volver a incluir reverses it, also with a reason.

Without a browser, do the same two steps, in this order, with a dashboard user that has `power_ens_admin` or `all_access`. Run the first one on the manager, or on the master in a cluster:

```bash
sudo /var/ossec/bin/agent_groups -a -i 001 -g ens-excluir-op.acc.1 -q
curl -sk -u "$DASH_USER" -H 'osd-xsrf: true' -H 'Content-Type: application/json' \
  -X POST "https://<dashboard>/api/powerEns/exclusions" \
  -d '{"agentId":"001","controlId":"op.acc.1","action":"EXCLUDED","reason":"<reason given by the organisation>"}'
```

If the record fails, undo the group change with `agent_groups -r -i 001 -g ens-excluir-op.acc.1 -q`, so that no exclusion exists without its reason. To include the machine again, remove it from the group and record `"action":"INCLUDED"` with a reason. The reason holds up to 1000 characters: write who decided and why, not only that the control does not apply.

## 6. Check the effect

Wait for the next engine pass, about 15 minutes, and read the verdicts again. Each change must have the effect you expected, and nothing else may have changed. If a change had another effect, report it and ask before reverting.

## 7. Report

Report in this order:

1. The changes and exclusions you made, from the record of rule 7.
2. The verdicts that still fail, grouped: system faults to fix, parameters the organisation must still declare, collection faults, cases for manual review.
3. What you could not do, and why.
