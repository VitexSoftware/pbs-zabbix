# pbs-zabbix

![pbs-zabbix icon](pbs-zabbix.svg)

Zabbix monitoring for [Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server)
(PBS). Alerts when a scheduled backup did not complete successfully — including
the case where it never even started (e.g. the network path between the
hypervisor and the PBS datastore was down).

## How it works

`pbs-zabbix-backup-check` reads Proxmox Backup Server task logs directly from
`/var/log/proxmox-backup/tasks`. Those files are world-readable, so the tool
needs no root privileges and does not call the PBS API.

It supports three modes, wired into Zabbix Agent 2 via
[`zabbix/pbs-backup.conf`](zabbix/pbs-backup.conf):

- `discovery` — Zabbix low-level discovery (LLD): lists `{#VMID}` for every
  VM/CT that has at least one recorded backup task.
- `age <vmid>` — seconds since the last **successful** (`TASK OK`) backup of
  `vmid`. Returns a large sentinel value if no successful backup was ever
  recorded, so a staleness trigger fires immediately.
- `status <vmid>` — the last recorded task line for `vmid` (success or
  failure), for alert context.

## Installing

```sh
apt install pbs-zabbix
systemctl restart zabbix-agent2
```

## Zabbix setup

On the Zabbix server, create (once the package is installed on the PBS host):

1. A discovery rule using item key `pbs.backup.discovery`.
2. Item prototypes:
   - `pbs.backup.age[{#VMID}]` (Numeric, unsigned, units `s`)
   - `pbs.backup.laststatus[{#VMID}]` (Text)
3. A trigger prototype, e.g.:

   ```
   last(/<host>/pbs.backup.age[{#VMID}])>{$PBS.BACKUP.MAXAGE:"{#VMID}"}
   ```

4. A user macro `{$PBS.BACKUP.MAXAGE}` with a sensible default (e.g. `108000`
   for 30 hours to cover a nightly backup schedule with some slack), and
   context overrides (`{$PBS.BACKUP.MAXAGE:"<vmid>"}`) for VMs/CTs on a
   different backup cadence (e.g. weekly jobs).

## License

MIT — see [LICENSE](LICENSE).
