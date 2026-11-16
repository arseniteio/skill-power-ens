# Wazuh topology

`install.sh server` decides what to install from what it finds on the host. Find the same facts yourself before the plan, so that the plan shows each host's role and the operator can correct it.

## Find each host's role

Run on each Wazuh host:

```bash
test -x /var/ossec/bin/wazuh-analysisd && echo manager
test -d /usr/share/wazuh-dashboard && echo dashboard
test -d /usr/share/wazuh-indexer && echo indexer
sudo /var/ossec/bin/wazuh-control info
grep -m1 '"version"' /usr/share/wazuh-dashboard/package.json
```

A host can have several roles. On a manager, read the `<cluster>` section of `/var/ossec/etc/ossec.conf`:

```bash
sudo sed -n '/<cluster>/,/<\/cluster>/p' /var/ossec/etc/ossec.conf
sudo /var/ossec/bin/cluster_control -l
```

The manager is a worker only when `<disabled>` is `no` and `<node_type>` is `worker`. Any other manager is the master or a single node. `cluster_control -l` lists the nodes with their type, or fails when the cluster is disabled. A deployment has one master; if you find none or two, stop and ask.

## What the installer does on each role

| Role | Installs | Restarts |
| --- | --- | --- |
| Master manager or single node | Rules 119000 to 119004, the `ens` group, the audit of the manager itself, the engine and its users | `wazuh-manager` |
| Worker manager | Waits for the rules the cluster syncs from the master, then loads them | `wazuh-manager` |
| Dashboard | The Power ENS plugin for its OpenSearch Dashboards version | `wazuh-dashboard` |
| Indexer only | Nothing. Do not run the installer there | None |

A host with several roles gets the parts of each, in one run. The engine runs only on the master: one engine per deployment.

On the master, the installer reads the indexer URLs from `/etc/filebeat/filebeat.yml` and the indexer CA from `/etc/filebeat/certs/root-ca.pem` or `/etc/wazuh-indexer/certs/root-ca.pem`. The master must reach every indexer on 9200. When the master has no indexer admin certificate, the installer asks for the indexer `admin` password.

Worker managers are not audited by detection. Only the master audits itself.

## Order

1. The master. Nothing else works until its run ends.
2. Each worker, one at a time. Wait until `cluster_control -l` on the master lists it again before the next.
3. Each dashboard, one at a time, so that a load balancer always has one serving.
4. The audited machines, with `install.sh agent --group ens`.

Upgrades follow the same order. Uninstall goes the other way: audited machines, dashboards and workers, then the master.

## Checks per host

| Host | Check |
| --- | --- |
| Master | The installer ends with `Done on this node`. `systemctl is-active oscompliance-engine` is `active`. `journalctl -u oscompliance-engine` shows a `pass` with `verdicts` and no `error` |
| Worker | The installer prints `rules loaded on this worker`. `/var/ossec/etc/rules/oscompliance_facts_rules.xml` exists. `cluster_control -l` on the master lists the worker |
| Dashboard | `systemctl is-active wazuh-dashboard` is `active`. The dashboard opens Power ENS |
| Audited machine | `agent_groups -l -g ens` on the master lists it within 15 minutes. It has `rule.id:119001` alerts |
