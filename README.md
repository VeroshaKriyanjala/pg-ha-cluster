# PostgreSQL 16 HA cluster — Patroni + etcd + HAProxy + Keepalived

One `ansible-playbook` run brings up a fully automatic high-availability
PostgreSQL cluster. Clients connect to a single floating VIP; failover is
handled by Patroni via an etcd quorum, and the load-balancer tier is made
redundant with Keepalived.

This is an Ansible implementation of the "Setting Up a PostgreSQL Cluster with
a Load Balancer using etcd, Patroni, HAProxy and Keepalived" guide, with two
additions:

* **Runs on RHEL 9 *and* Ubuntu** — every role branches on `ansible_os_family`.
* **Per-server hardware specs drive the config** — each node's CPU, memory and
  storage are declared in the inventory and turned into PostgreSQL settings
  automatically (see [Sizing / auto-tuning](#sizing--auto-tuning)).

## Topology

| Host        | IP              | Role                                   |
|-------------|-----------------|----------------------------------------|
| postgres-1  | 172.15.100.131  | PostgreSQL 16 + Patroni + etcd (etcd1) |
| postgres-2  | 172.15.100.132  | PostgreSQL 16 + Patroni + etcd (etcd2) |
| postgres-3  | 172.15.100.133  | PostgreSQL 16 + Patroni + etcd (etcd3) |
| haproxy-1   | 172.15.100.18   | HAProxy + Keepalived (MASTER, prio 101)|
| haproxy-2   | 172.15.100.19   | HAProxy + Keepalived (BACKUP, prio 100)|
| **VIP**     | 172.15.100.17   | floating — clients connect here        |

### Connection endpoints (via the VIP)

| Port | Purpose                              |
|------|--------------------------------------|
| 5000 | read-write — always the leader       |
| 6000 | read-only  — replicas (round-robin)  |
| 7000 | HAProxy stats web page               |
| 8008 | Patroni REST API (health checks)     |
| 5432 | PostgreSQL (direct, per node)        |
| 2379 / 2380 | etcd client / peer            |

```bash
# writes / migrations
psql "host=172.15.100.17 port=5000 user=postgres dbname=postgres"
# read-only queries
psql "host=172.15.100.17 port=6000 user=postgres dbname=postgres"
```

HAProxy routes port 5000 using Patroni's REST health checks (`GET /primary`
returns 200 only on the leader; `GET /replica` returns 200 on running
replicas), so the write port always follows the current leader with no
client-side change.

## Layout

```
ansible.cfg              # config: inventory, become, collections path
requirements.yml         # ansible.posix + community.general
site.yml                 # master playbook (ordered plays)
verify.yml               # post-deploy health check
inventory/hosts.yml      # hosts + per-server_specs (cpu/memory/storage)
group_vars/
  all.yml                # the main file you edit: versions, VIP, tuning, creds
  pgcluster.yml          # firewall ports for DB nodes
  haproxy.yml            # firewall ports + sizing for LB nodes
roles/
  common/                # base pkgs, SELinux, firewall, data-disk format+mount
  etcd/                  # etcd DCS (RPM on RHEL / tarball on Ubuntu)
  postgresql/            # PGDG repo + PostgreSQL 16 install
  patroni/               # venv install, TLS, auto-tuned config, watchdog, systemd
  haproxy/               # TCP load balancer
  keepalived/            # unicast VRRP floating VIP
```

## Sizing / auto-tuning

Each host carries a `server_specs` block in `inventory/hosts.yml`:

```yaml
postgres-1:
  ansible_host: 172.15.100.131
  etcd_name: etcd1
  patroni_name: postgresql-01
  server_specs:
    cpu_cores: 4
    memory_mb: 16384          # 16 GB
    storage:
      device: /dev/sdb        # OMIT to skip disk management
      fstype: xfs
      size_gb: 100            # informational
```

`cpu_cores` and `memory_mb` are the **single source of truth** for the
PostgreSQL resource settings. On each node the `patroni` role computes them
from the formulas in `group_vars/all.yml`:

| Parameter                          | Formula (defaults)                    | 4 vCPU / 16 GB result |
|------------------------------------|---------------------------------------|-----------------------|
| `shared_buffers`                   | 25% of RAM                            | 4096 MB               |
| `effective_cache_size`             | 75% of RAM                            | 12288 MB              |
| `maintenance_work_mem`             | RAM / 16, capped at 2048 MB           | 1024 MB               |
| `wal_buffers`                      | shared_buffers / 32, capped at 16 MB  | 16 MB                 |
| `work_mem`                         | (25% RAM) / max_connections, min 4 MB | ~20 MB                |
| `max_worker_processes`             | cpu_cores                             | 4                     |
| `max_parallel_workers`             | cpu_cores                             | 4                     |
| `max_parallel_workers_per_gather`  | cpu_cores / 2, min 1                  | 2                     |
| `max_parallel_maintenance_workers` | cpu_cores / 2, min 1                  | 2                     |

The `work_mem` default treats "25% of RAM" as a pool divided across
`max_connections`, so the naive worst case stays inside that pool. Complex
queries can allocate `work_mem` several times over, so lower `max_connections`
(or set `work_mem` explicitly) for heavy analytical workloads.

**Three ways to change the numbers**, lowest to highest precedence:

1. Adjust the fractions/caps in `group_vars/all.yml`
   (`shared_buffers_fraction`, `effective_cache_fraction`, …).
2. Set `postgres_max_connections` (also feeds HAProxy's per-server `maxconn`).
3. Force any final GUC via `postgres_parameters_override`, cluster-wide in
   `group_vars/all.yml` or per node in `host_vars/<host>.yml`:

   ```yaml
   postgres_parameters_override:
     random_page_cost: 1.1     # SSD storage
     work_mem: "32MB"
     jit: "off"
   ```

Give different nodes different specs and they get tuned independently — a
larger leader and smaller replicas is fine.

### Storage

If a host's `server_specs.storage.device` is set, the `common` role creates
the filesystem (only if the device is blank — it never reformats an existing
one) and mounts it, persisted in `/etc/fstab`, at the PostgreSQL data parent
(`/var/lib/pgsql` on RHEL, `/var/lib/postgresql` on Ubuntu; override with
`storage.mount`). **Omit the `storage` block entirely** and the role simply
makes sure the data directory exists — provision disks however you like.

## Prerequisites

- A control machine with `ansible-core` (>= 2.15).
- Two Ansible collections:

  ```bash
  ansible-galaxy collection install -r requirements.yml -p collections
  ```

- SSH access to all five hosts as a sudo-capable user.
- Targets running **RHEL/Rocky/Alma 9** or **Ubuntu 20.04 / 22.04 / 24.04**
  (a mixed inventory works — each role adapts).
- Outbound internet on the targets (PGDG repo, PyPI, etcd release).

## Configure

1. Edit `inventory/hosts.yml` — set `ansible_user`, the IPs, and each node's
   `server_specs` (cpu / memory / storage).
2. Edit `group_vars/all.yml` — set the VIP, and **change every password** plus
   `vrrp_auth_pass`. Then encrypt it:

   ```bash
   ansible-vault encrypt group_vars/all.yml
   ```

3. Optional feature toggles in `group_vars/all.yml`: `patroni_enable_ssl`,
   `patroni_watchdog_enabled`, `manage_firewall`, `manage_selinux`,
   `selinux_state`, `manage_sysctl`. For ARM targets set `etcd_arch: arm64`.

## Deploy

```bash
# test connectivity
ansible all -m ping

# full run (add --ask-vault-pass if you encrypted group_vars)
ansible-playbook site.yml
```

Plays run in the required order: base setup → etcd quorum → PostgreSQL install
→ Patroni bootstrap → HAProxy + Keepalived.

## Verify

```bash
ansible-playbook verify.yml           # asserts exactly one running leader

# or manually, on any DB node:
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

`patronictl list` should show one `Leader` and two `Replica` rows, all
`running`. HAProxy stats: `http://172.15.100.17:7000/`.

## Test failover

```bash
# graceful: hand the leader role to another node
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover

# hard: stop Patroni on the leader and watch a replica get promoted
sudo systemctl stop patroni

# confirm the write port followed the new leader
psql -h 172.15.100.17 -p 5000 -U postgres \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"
```

Stopping HAProxy on the MASTER LB node moves the VIP to the BACKUP
(`sudo systemctl stop haproxy`), so the LB tier survives a node loss too.

## Notes & caveats

- **etcd needs a majority.** This 3-node cluster survives one node failure;
  never run it on an even number of nodes, and for production spread the three
  members across separate failure domains.
- **DCS settings are written once.** `patroni.yml`'s `bootstrap.dcs` block is
  applied only at first cluster init. Change it later with
  `patronictl edit-config`, not by re-editing the file (re-running the play
  updates the on-disk file but Patroni ignores `bootstrap` after init).
- **etcd on Ubuntu** installs upstream release binaries (`etcd_version`) into
  `/usr/local/bin`; on RHEL it installs the PGDG RPM (`etcd_rpm_url`). Both
  read the same `/etc/etcd/etcd.conf` environment file.
- **SELinux** defaults to `permissive` (logs, never blocks) rather than a full
  disable; set `selinux_state: disabled` to match the original guide exactly.
- **Ubuntu firewall**: ufw is left *inactive* (rules are added but not enabled)
  to avoid locking out SSH. Enable it yourself once you've confirmed the rules,
  and add an iptables rule for the VRRP protocol between the LB nodes.
- Self-signed TLS certs are generated per node under `/etc/patroni/`. Replace
  them with your own CA-signed certs for production, or set
  `patroni_enable_ssl: false`.
