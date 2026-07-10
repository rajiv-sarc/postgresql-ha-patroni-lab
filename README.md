# PostgreSQL HA Lab — Patroni + etcd (Docker Compose)

A local, self-contained 2-node PostgreSQL high-availability cluster built with
[Patroni](https://patroni.readthedocs.io/) (via Zalando's Spilo image) and
[etcd](https://etcd.io/) as the distributed consensus store (DCS). Used to
validate leader election, automated failover, and cluster health monitoring
via Patroni's REST API.

## Architecture

- **etcd** — single-node distributed key-value store acting as the DCS that
  Patroni uses for leader election and cluster state.
- **patroni1 / patroni2** — two PostgreSQL 15 nodes managed by Patroni
  (Spilo image), both members of the same cluster scope (`pg-ha`), each
  exposing:
  - PostgreSQL on `5432` (patroni1) / `5433` (patroni2) on the host
  - Patroni REST API on `8008` (patroni1) / `8009` (patroni2) on the host,
    used for health checks and cluster status

## Prerequisites

- Docker and Docker Compose installed locally

## Running the cluster

```bash
docker compose up -d
docker compose ps
```

Check cluster status via the Patroni REST API:

```bash
curl http://localhost:8008/cluster | jq
```

Or, if you have `patronictl` available (e.g. exec into a container):

```bash
docker exec -it patroni1 patronictl -c /config/patroni.yml list
```

This shows which node is the current **Leader**, replication lag, and
timeline (TL) for each member.

## Testing failover

1. Confirm which node is currently the leader:
   ```bash
   docker exec -it patroni1 patronictl -c /config/patroni.yml list
   ```
2. Stop the leader container to simulate a failure:
   ```bash
   docker stop patroni1   # or patroni2, whichever is the leader
   ```
3. Watch the remaining node get promoted:
   ```bash
   docker exec -it patroni2 patronictl -c /config/patroni.yml list
   ```
4. Time how long it takes between stopping the leader and the standby
   reporting `Leader` status — this is your real-world failover time.
5. Restart the stopped node and confirm it rejoins as a replica:
   ```bash
   docker start patroni1
   docker exec -it patroni2 patronictl -c /config/patroni.yml list
   ```

## Notes

- This is a **local lab / self-study project**, not a production deployment.
  Passwords in `patroni/*.yml` are placeholders for local testing only —
  never reuse these values outside a local, non-networked environment.
- `ETCD_LOCAL_NODE` and volume mounts are commented/adjusted for a
  single-etcd-node local setup; a production cluster would run etcd (or
  Consul/ZooKeeper) as a proper multi-node quorum.

## What this demonstrates

- Patroni cluster bootstrapping and configuration via `patroni.yml`
- etcd as a DCS for leader election
- Automated failover and replica rejoin behavior
- Patroni's REST API for cluster health/status monitoring
