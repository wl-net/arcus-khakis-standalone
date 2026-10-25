# arcus-khakis-single

Docker Compose configuration for a single-node Arcus Platform backend: one
Cassandra node, one ZooKeeper, and one Kafka broker. The multi-DC version is
[arcus-khakis-cluster](https://github.com/wl-net/arcus-khakis-cluster).

```bash
docker compose -f docker/docker-compose.yml up -d
```

## Upgrading from a pre-2026.10 compose file

ZooKeeper's transaction logs (`/datalog`) used to live on an anonymous volume
and now have a named one. Copy them across on the first deploy, or ZooKeeper
comes back with only its last snapshot and drops everything written since:

```bash
C="docker compose -f docker/docker-compose.yml"
datalog() { docker inspect -f '{{range .Mounts}}{{if eq .Destination "/datalog"}}{{.Name}}{{end}}{{end}}' "$1"; }

OLD=$(datalog "$($C ps -q zookeeper-0)")
$C stop kafka-0 zookeeper-0
$C up --no-start zookeeper-0   # creates the named volume, doesn't start ZK
NEW=$(datalog "$($C ps -aq zookeeper-0)")
docker run --rm -v "$OLD":/old:ro -v "$NEW":/new busybox cp -a /old/. /new/
$C up -d
```

The Cassandra image moved from 3.11 to 4.0 in early 2026. If the host's
`latest` predates that, pull the new one and rewrite the SSTables once it's
back up:

```bash
docker compose -f docker/docker-compose.yml exec cassandra-0 nodetool -h ::ffff:127.0.0.1 upgradesstables
```
