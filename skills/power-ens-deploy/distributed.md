# Distributed deployments

The guides cover all-in-one and single-node Docker. In any other layout, follow the sections of `oscompliance-ens/docs/en/install.md` on the host each one belongs to. Tell the operator that the layout is outside the guides before you start, and record every adaptation.

`install.sh server` refuses a host without the dashboard. Use the release package on each host instead: download the tarball of its architecture and `SHA256SUMS`, check them with `sha256sum -c --ignore-missing SHA256SUMS`, extract it and run the commands from that folder. On dashboard hosts, also download the plugin zip, `deploy-plugin.sh` and `create-admin-role.sh` of the same release, as in section 1.

## Engine host

Choose one host that reaches the indexer on 9200 and the master's API on 55000, usually an indexer node. It needs the indexer admin certificates for `engine-users.sh`.

## Order

1. Rules, section 2, on every manager node, master and workers. Run `wazuh-analysisd -t` on each. Restart one node at a time, and wait until `cluster_control -l` lists it again before the next.
2. Group, section 2, on the master only: create `ens` and add the machines. The cluster copies the group configuration to the workers.
3. Engine users, section 3, on the engine host, with the indexer and the master API:

   ```bash
   sudo --preserve-env=API_ADMIN_PASSWORD,OSC_INDEXER_PASSWORD,OSC_WAZUH_API_PASSWORD \
     INDEXER_URL=https://<indexer>:9200 API_URL=https://<master>:55000 \
     bash wazuh/engine-users.sh create
   ```

   `CERTS` defaults to `/etc/wazuh-indexer/certs`; set it if the admin certificates are elsewhere.
4. Plugin, section 4, on every dashboard, one at a time, so the load balancer always has one serving. Check each with `systemctl is-active wazuh-dashboard`.
5. Configuration users: run `create-admin-role.sh --map <user>` on the engine host, with `OS_URL=https://<indexer>:9200`.
6. First save in the dashboard, as in section 5.
7. Engine, section 6, on the engine host. Copy the master's `/var/ossec/api/configuration/ssl/server.crt` to the engine host. Before you enable the service, edit `ExecStart` in `/etc/systemd/system/oscompliance-engine.service`: `-indexer` to the indexer URL, `-ca` to the indexer root CA on that host, `-wazuh-api` to the master API URL and `-wazuh-api-ca` to the copied certificate. Keep `-wazuh-api-server-name` as the name in that certificate, `localhost` by default.
8. Managers, section 8, on every manager node the operator wants audited. Each one is a machine of its own.
9. Audited machines with `install.sh agent`, as in the skill.

## Checks

- `journalctl -u oscompliance-engine` on the engine host shows `group configuration` with `applied` true and a `pass` with no `error`.
- `/var/ossec/etc/shared/ens/agent.conf` on the master and on each worker holds the `oscompliance begin` block.
- Each dashboard opens Power ENS.
