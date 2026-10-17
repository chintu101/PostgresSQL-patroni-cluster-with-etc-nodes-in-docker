# PostgresSQL-patroni-cluster-with-etc-nodes-in-docker
setup to run a postgresSQL cluster with etc nodes in docker, goal is to create a system that showcases basic multi-node architecture and how patroni with etcd handles concurrency during node failures
<br><br>
docker setup was done in ubuntu. I don't think it is necessary to drop a windows guide, ill add it later if i have to
<br><br>
# Project Structure
```text
pg-lab/
├── docker-compose.yml        # defines the etc and pg+patroni nodes
└── patroni/
    ├── Dockerfile            # installed a postgres 16 image with patroni
    └── patroni.yml.tpl       # generates patroni.yml from env vars, then starts Patroni
```
## References

### etcd

- [etcd documentation](https://etcd.io/docs/)
- [etcd GitHub repository](https://github.com/etcd-io/etcd)
- [etcd clustering guide](https://etcd.io/docs/latest/op-guide/clustering/)
- [etcd command-line flags](https://etcd.io/docs/latest/op-guide/configuration/)
- [Raft consensus algorithm (paper)](https://raft.github.io/)

### Patroni

- [Patroni documentation](https://patroni.readthedocs.io/)
- [Patroni GitHub repository](https://github.com/patroni/patroni)
- [Patroni configuration reference](https://patroni.readthedocs.io/en/latest/patroni_configuration.html)
- [Patroni REST API](https://patroni.readthedocs.io/en/latest/rest_api.html)
- [patronictl command reference](https://patroni.readthedocs.io/en/latest/patronictl.html)

### PostgreSQL

- [PostgreSQL 16 documentation](https://www.postgresql.org/docs/16/)
- [High availability, load balancing, and replication](https://www.postgresql.org/docs/16/high-availability.html)
- [Streaming replication](https://www.postgresql.org/docs/16/warm-standby.html)
- [pg_hba.conf reference](https://www.postgresql.org/docs/16/auth-pg-hba-conf.html)


# Setup
Creating the directory to work in
```bash
mkdir -p ~/pg-lab/patroni
cd ~/pg-lab
```

Make sure docker is installed with the compose plugin
```bash
# Install Docker Engine and the Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Confirm that compose plugin has been added
```bash
docker compose version
```

The following instructions is to setup the simplest form of this system with 3 etcd nodes and 5 pg+patroni nodes
<br>
This is the Dockerfile:
```bash
cat > Dockerfile <<'EOF'
# Official PostgreSQL 16 image (Debian-based), plus Patroni in a virtualenv.
# A venv is used because Debian's system Python blocks global pip installs.
FROM postgres:16

USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 python3-venv \
    && python3 -m venv /opt/patroni \
    && /opt/patroni/bin/pip install --no-cache-dir "patroni[etcd3]" psycopg2-binary \
    && rm -rf /var/lib/apt/lists/*

COPY entrypoint.sh /usr/local/bin/entrypoint-patroni.sh
RUN chmod +x /usr/local/bin/entrypoint-patroni.sh

USER postgres
ENTRYPOINT ["/usr/local/bin/entrypoint-patroni.sh"]
EOF
```
Creating the entrypoint script entrypoint.sh
<br>
This writes the patroni.yml from NODE_NAME and the password env variables, then starts up patroni
```bash
cat > entrypoint.sh <<'EOF'
#!/bin/bash
set -e

cat > /tmp/patroni.yml <<YAML
scope: pg-lab
name: ${NODE_NAME}

restapi:
  listen: 0.0.0.0:8008
  connect_address: ${NODE_NAME}:8008

etcd3:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379          # Specified 3 etcd nodes here along with their ports
                                                   # you can add more based on how many pg nodes you are going to setup
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 100
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host replication replicator all md5
    - host all all all md5
  users:
    admin:
      password: admin
      options: [createrole, createdb]

postgresql:
  listen: 0.0.0.0:5432
  connect_address: ${NODE_NAME}:5432
  data_dir: /var/lib/postgresql/data/pgdata
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    superuser:
      username: postgres
      password: ${POSTGRES_PASSWORD}
    replication:
      username: replicator
      password: ${REPLICATION_PASSWORD}
  parameters:
    unix_socket_directories: '/tmp'
YAML

exec /opt/patroni/bin/patroni /tmp/patroni.yml
EOF
```

Build the image from dir
```bash
docker build -t pg-lab-patroni ./patroni
```
in case the image for some reason does not add executable perms for entrpoint.sh
<br>
Run
```bash
chmod +x entrypoint.sh
```
After build, check if patroni is installed.
```bash
docker run --rm --entrypoint /opt/patroni/bin/patroni pg-lab-patroni --version
```
Now creating the docker-compose.yml
```bash
cat > docker-compose.yml <<'EOF'
# followed by the compose file, the file is downloadable from the repo too.
```
# docker-compose.yml explained
docker-compose.yml
This file defines the lab's eight containers: three etcd nodes and five Postgres + Patroni nodes, all on one shared Docker network.

## Project name

```yaml
name: pg-lab
```

Sets the project name. Compose uses it as a prefix for container, network, and volume names (for example, `pg-lab-pg1-1`), and groups everything under one project for commands like `docker compose down`.

## Shared template for Postgres nodes

```yaml
x-pg-common: &pg-common
  build: ./patroni
  image: pg-lab-patroni
  ...
```

- The `x-` prefix marks a top-level key that Compose ignores as a service. It exists only to hold reusable blocks.
- `&pg-common` is a YAML **anchor**, a name for this block that other places can refer to.
- `build: ./patroni` builds the image from the Dockerfile in `patroni/`.
- `image: pg-lab-patroni` names the built image, so all five nodes share one image instead of building five times.
- `depends_on` makes Compose start the etcd containers first. It controls start order only, not readiness, so Patroni may still retry until etcd is ready.
- `networks: [lab]` attaches the nodes to the shared network.
- `restart: unless-stopped` restarts a container after a crash or daemon restart, but not after you stop it manually.
- `shm_size: 256mb` raises the shared memory limit inside the container. Postgres uses shared memory for its buffers, and the default is too small for some workloads.

## Shared environment for Postgres nodes

```yaml
environment: &pg-env
  POSTGRES_PASSWORD: lab-super-pw
  REPLICATION_PASSWORD: lab-repl-pw
```

Another anchor, this time for environment variables. `entrypoint.sh` reads these to fill in the Patroni config. `POSTGRES_PASSWORD` becomes the superuser password, and `REPLICATION_PASSWORD` is the password replicas use to connect to the primary.

## etcd nodes

### Base definition (etcd1)

```yaml
etcd1: &etcd
  image: quay.io/coreos/etcd:v3.5.15
  hostname: etcd1
```

- `image` uses the official etcd image at a pinned version.
- `hostname: etcd1` sets the container's hostname. Other containers reach it by this name.
- `&etcd` anchors the service so etcd2 and etcd3 can reuse it.

```yaml
command:
  - etcd
  - --name=etcd1
  - --data-dir=/etcd-data
  - --initial-advertise-peer-urls=http://etcd1:2380
  - --listen-peer-urls=http://0.0.0.0:2380
  - --listen-client-urls=http://0.0.0.0:2379
  - --advertise-client-urls=http://etcd1:2379
  - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
  - --initial-cluster-state=new
  - --initial-cluster-token=pg-lab-etcd
```

These are etcd's startup flags:

- `--name` is this member's identity in the cluster.
- `--data-dir` is where Raft logs and snapshots are stored inside the container.
- `--listen-peer-urls` is where etcd listens for other etcd members (port 2380). `0.0.0.0` means all interfaces.
- `--listen-client-urls` is where etcd listens for clients, including Patroni (port 2379).
- `--initial-advertise-peer-urls` and `--advertise-client-urls` are the addresses other members and clients use to reach this node.
- `--initial-cluster` lists all three members, so each one knows the full membership at bootstrap.
- `--initial-cluster-state=new` marks this as a fresh cluster, not one rejoining.
- `--initial-cluster-token` is a shared name that keeps this cluster separate from any other etcd cluster on the network.

### etcd2 and etcd3

```yaml
etcd2:
  <<: *etcd
  hostname: etcd2
  command: ...
```

`<<: *etcd` merges the base definition into this service. Each node overrides only what differs: its hostname and its `--name` and advertise URLs. The `command` list is repeated in full, because YAML merges replace whole keys rather than merging list items.

## Postgres nodes (pg1 to pg5)

```yaml
pg1:
  <<: *pg-common
  hostname: pg1
  environment:
    <<: *pg-env
    NODE_NAME: pg1
  ports: ["5441:5432", "8081:8008"]
```

- `<<: *pg-common` brings in the shared build, network, and restart settings.
- `hostname: pg1` gives the node its name. Patroni uses this name for `connect_address`, and the etcd host list refers to the etcd nodes by name in the same way.
- `environment` merges in the shared passwords (`<<: *pg-env`) and adds `NODE_NAME`, which `entrypoint.sh` uses as the Patroni node name.
- `ports` maps host ports to container ports. `5441:5432` means port 5441 on your laptop reaches Postgres (5432) inside pg1. `8081:8008` does the same for the Patroni REST API. Each node gets its own host ports so you can reach them separately.

## Network

```yaml
networks:
  lab:
    driver: bridge
```

Creates a private network named `lab` that all eight containers join. The `bridge` driver is Docker's standard virtual network. Containers resolve each other by hostname here, which is why `etcd1:2379` works without IP addresses.

## Why it's structured this way

The file uses YAML anchors (`&` to define, `*` to reference, `<<:` to merge) so the five Postgres nodes and three etcd nodes don't repeat the same settings. The only per-node differences are the hostname, the node name, and the host ports, which keeps the file short and makes the variation between nodes easy to see.

## Validating the file
```bash
docker compose config --quiet && echo "compose file OK"
```
## Confirming the service have been defined
```bash
docker compose config --services
```
It should output all eight names of the nodes.
<br>
so etcd1, etcd2, etcd3 and pg 1 to 5

# Starting the nodes
start etcd then the pg nodes
## Start the three etcd nodes
```bash
docker compose up -d etcd1 etcd2 etcd3
docker compose ps
```
it should shows that all the etcd nodes as "running"

## Check that the etcd nodes have a leader and a quorum
```bash
docker compose exec etcd1 etcdctl --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 endpoint status --write-out=table
```
You should see three rows. One row should have true in the IS LEADER column

## Start the pg nodes
```bash
docker compose up -d --build pg1 pg2 pg3 pg4 pg5
docker compose ps
```
The first build takes several minutes because it installs Patroni. Once it finishes, all eight containers should show running.

## View the topology of the 5 pg nodes
```bash
docker compose exec pg1 /opt/patroni/bin/patronictl -c /tmp/patroni.yml list
```
# Connecting to the leader node
The leader is whichever node Patroni reports. Ports 5441–5445 map to pg1–pg5, so if pg1 is leader, use 5441.
<br>
Connect with `psql`. if you don't have it install it with the command
```bash
sudo apt-get install -y postgresql-client
```
Connect to the leader:
```bash
psql -h localhost -p 5441 -U postgres           # Change the port number to whatever is the leader node
                                                # at this instance my leader node was at 5441
```
Enter the password `lab-super-pw`. Then run:
```sql
SELECT pg_is_in_recovery();
```
It returns `f` on the leader and `t` on a replica, showing whether the node is in recovery.
## A small test
On the leader, create a table and insert a row:

```sql
CREATE TABLE lab_check (id int primary key, note text);
INSERT INTO lab_check VALUES (1, 'written on leader');
\q
```

Read the row from a replica (for example, port 5443 if pg3 is a replica):

```bash
psql -h localhost -p 5443 -U postgres -c "SELECT * FROM lab_check;"
```

You should see the row.

Confirm the replica is read-only:

```bash
psql -h localhost -p 5443 -U postgres -c "INSERT INTO lab_check VALUES (2, 'should fail');"
```

This should fail with `cannot execute INSERT in a read-only transaction`.

## Troubleshooting: password prompts

The superuser password is `lab-super-pw`. Replicas use the same authentication.

If you see `password authentication failed`, check the password against the leader:

```bash
PGPASSWORD=lab-super-pw psql -h localhost -p 5441 -U postgres -c "SELECT 1;"
```

If that still fails, search the logs for password or authentication errors:

```bash
docker compose logs pg1 pg2 pg3 pg4 pg5 | grep -i -E "password|authentication"
```

### Clean reset

This is a throwaway lab, so the simplest fix is to delete all containers and volumes and rebuild:

```bash
cd ~/pg-lab
docker compose down -v
docker compose up -d etcd1 etcd2 etcd3
docker compose up -d --build pg1 pg2 pg3 pg4 pg5
```

Wait about a minute for leader election, then run `patronictl list` again.

## Why `psql` didn't prompt

`psql` only asks for a password when the server requires one and none was supplied. The `PGPASSWORD` environment variable supplies it, so no prompt appears. The server still checks the password: the `md5` rules in `pg_hba` reject connections without the correct one.

To see the prompt, run the command without the variable:

```bash
psql -h localhost -p 5441 -U postgres -c "SELECT 1;"
```

To avoid typing the password each time, store it in `~/.pgpass`:

```bash
echo "localhost:5441:*:postgres:lab-super-pw" >> ~/.pgpass
echo "localhost:5442:*:postgres:lab-super-pw" >> ~/.pgpass
echo "localhost:5443:*:postgres:lab-super-pw" >> ~/.pgpass
echo "localhost:5444:*:postgres:lab-super-pw" >> ~/.pgpass
echo "localhost:5445:*:postgres:lab-super-pw" >> ~/.pgpass
chmod 600 ~/.pgpass
```
# Watch the leader election upon node failure
replace pg1 with whichever is your leader node.
```bash
docker compose stop pg1          # my instance leader was pg1, change to whichever node is leader
```
## Wait for a few seconds for pg ttl, so the old leader key expires before a replica can take it
```bash
docker compose exec pg2 /opt/patroni/bin/patronictl -c /tmp/patroni.yml list
```
Check that the data survived. Use the new leader’s port, which is 5441 to 5445 depending on the node:
```bash
psql -h localhost -p 5442 -U postgres -c "SELECT * FROM lab_check;"          # new leader was pg2
                                                                             # hence the port 5442
```

## Confirm that write works on the new leader
```bash
psql -h localhost -p 5442 -U postgres -c "INSERT INTO lab_check VALUES (3, 'written after failover');"
```
## Restart the stop node and watch it come back as a replica
```bash
docker compose start pg1
sleep 20
docker compose exec pg2 /opt/patroni/bin/patronictl -c /tmp/patroni.yml list
```
