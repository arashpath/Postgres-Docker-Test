# PostDock - Postgres + Docker
PostgreSQL cluster with **High Availability** and **Self Healing** features for any cloud and docker environment ( Docker Compose, Docker Swarm )
- Hacked from [PostDock Orginal Repo](https://github.com/paunin/PostDock)

![Formula](https://raw.githubusercontent.com/paunin/PostDock/master/artwork/formula2.png)


- [PostDock - Postgres + Docker](#postdock---postgres--docker)
  - [Info](#info)
    - [Features](#features)
    - [What's in the box](#whats-in-the-box)
  - [Start cluster with docker-compose](#start-cluster-with-docker-compose)
  - [Configuring the cluster](#configuring-the-cluster)
    - [Postgres](#postgres)
    - [Pgpool](#pgpool)
    - [Barman](#barman)
    - [Other configurations](#other-configurations)
  - [Adaptive mode](#adaptive-mode)
  - [SSH access](#ssh-access)
  - [Replication slots](#replication-slots)
  - [Extended version of postgres](#extended-version-of-postgres)
  - [Backups and recovery](#backups-and-recovery)
  - [Health-checks](#health-checks)
  - [Useful commands](#useful-commands)
  - [Scenarios](#scenarios)
  - [Publications](#publications)
  - [How to contribute](#how-to-contribute)
  - [FAQ](#faq)
  - [Documentation and manuals](#documentation-and-manuals)

-------

## Info
### Features
* High Availability
* Self Healing and Automated Reconstruction
* [Split Brain](https://en.wikipedia.org/wiki/Split-brain_(computing)) Tolerance
* Eventually/Partially Strict/Strict Consistency modes
* Reads Load Balancing and Connection Pool
* Incremental backup (with optional zero data loss, [RPO=0](https://en.wikipedia.org/wiki/Recovery_point_objective))
* Semi-automated Point In Time Recovery Procedure
* Monitoring exporters for all the components(nodes, balancers, backup)

### What's in the box
This project includes:
* Dockerfiles for `postgresql` cluster and backup system
    * [postgresql](Postgres.Dockerfile)
    * [pgpool](Pgpool.Dockerfile)
    * [barman](Barman.Dockerfile)
* Examples of usage(suitable for production environment as architecture has fault protection with auto failover)
    * example of [docker-compose](docker-compose.yml) file to start this cluster.

## Start cluster with docker-compose

To start cluster run it as normal `docker-compose` application `docker-compose -f docker-compose.yml up -d pgmaster pgslave1 pgslave2 pgslave3 pgslave4 pgpool backup`

Schema of the example cluster:

```
node1 (primary)        --|
|  |- backup (backup)  --|
|                      --|----pgpool (master_slave_mode stream)
|- node2 (slave)       --|
|- node3 (slave)       --|
```

Each `postgres` node (`pgmaster`, `pgslaveX`) is managed by `repmgr/repmgrd`. It allows to use automatic `failover` and check cluster status.

Please check comments for each `ENV` variable in [docker-compose.yml](docker-compose.yml) file to understand parameter for each cluster node

## Configuring the cluster

You can configure any node of the cluster(`postgres.conf`) or pgpool(`pgpool.conf`) with ENV variable `CONFIGS` (format: `variable1:value1[,variable2:value2[,...]]`, you can redefine delimiter and assignment symbols by using variables `CONFIGS_DELIMITER_SYMBOL`, `CONFIGS_ASSIGNMENT_SYMBOL`). Also see the Dockerfiles and [docker-compose.yml](docker-compose.yml) files in the root of the repository to understand all available and used configurations!

### Postgres

For the rest - you better **follow** the advise and look into the [Postgres.Dockerfile](Postgres.Dockerfile) file - it full of comments :)

### Pgpool

The most important part to configure in Pgpool (apart of general `CONFIGS`) is backends and users which could access these backends. You can configure backends with ENV variable. You can find good example of setting up pgpool in [docker-compose.yml](docker-compose.yml) file:

```
DB_USERS: monkey_user:monkey_pass # in format user:password[,user:password[...]]
BACKENDS: "0:pgmaster:5432:1:/var/lib/postgresql/data:ALLOW_TO_FAILOVER,1:pgslave1::::,3:pgslave3::::,2:pgslave2::::" #,4:pgslaveDOES_NOT_EXIST::::
            # in format num:host:port:weight:data_directory:flag[,...]
            # defaults:
            #   port: 5432
            #   weight: 1
            #   data_directory: /var/lib/postgresql/data
            #   flag: ALLOW_TO_FAILOVER
REQUIRE_MIN_BACKENDS: 3 # minimal number of backends to start pgpool (some might be unreachable)
```

### Barman

The most important part for barman is to setup access variables. Example can be found in [docker-compose.yml](docker-compose.yml) file:

```
REPLICATION_USER: replication_user # default is replication_user
REPLICATION_PASSWORD: replication_pass # default is replication_pass
REPLICATION_HOST: pgmaster
POSTGRES_PASSWORD: monkey_pass
POSTGRES_USER: monkey_user
POSTGRES_DB: monkey_db
```

### Other configurations

**See the Dockerfiles and [docker-compose.yml](docker-compose.yml) files in the root of the repository to understand all available and used configurations!**


## Adaptive mode

'Adaptive mode' means that node will be able to decide if instead of acting as a master on it's start or switch to standby role.
That possible if you pass `PARTNER_NODES` (comma separated list of nodes in the cluster on the same level).
So every time container starts it will check if it was master before and if there is no new master around (from the list `PARTNER_NODES`),
otherwise it will start as a new standby node with `upstream = new master` in the cluster.

Keep in mind: this feature does not work for cascade replication and you should not pass `PARTNER_NODES` to nodes on second level of the cluster.
Instead of it just make sure that all nodes on the first level are running, so after restart any node from second level will be able to follow initial upstream from the first level.
That also can mean - replication from second level potentially can connect to root master... Well not a big deal if you've decided to go with adaptive mode.
But nevertheless you are able to play with `NODE_PRIORITY` environment variable and make sure entry point for second level of replication will never be elected as a new root master 


## SSH access

If you have need to organize your cluster with some tricky logic or less problematic cross checks. You can enable SSH server on each node. Just set ENV variable `SSH_ENABLE=1` (disabled by default) in all containers (including pgpool and barman). That will allow you to connect from any to any node by simple command under `postgres` user: `gosu postgres ssh {NODE NETWORK NAME}`

You also will have to set identical ssh keys to all containers. For that you need to mount files with your keys in paths `/home/postgres/.ssh/keys/id_rsa`, `/home/postgres/.ssh/keys/id_rsa.pub`.


## Replication slots

If you want to disable the feature of Postgres>=9.4 - [replication slots](https://www.postgresql.org/docs/9.4/static/catalog-pg-replication-slots.html) simply set ENV variable `USE_REPLICATION_SLOTS=0` (enabled by default). So cluster will rely only on Postgres configuration `wal_keep_segments` (`500` by default). You also should remember that default number for configuration `max_replication_slots` is `5`. You can change it (as any other configuration) with ENV variable `CONFIGS`.


## Extended version of postgres

Component `postgres-extended` from the section [Docker images tags convention](#docker-images-tags-convention) should be used if you want to have postgres with extensions and libraries. Each directory [here](pgsql/extensions/bin/extensions) represents extension included in the image.

## Backups and recovery

[Barman](http://docs.pgbarman.org/) is used to provide real-time backups and Point In Time Recovery (PITR)..
This image requires connection information(host, port) and 2 sets of credentials, as you can see from [the Dockerfile](Barman.Dockerfile):

* Replication credentials
* Postgres admin credentials

Barman acts as warm standby and stream WAL from source. Additionaly it periodicaly takes remote physical backups using `pg_basebackup`.
This allows to make PITR in reasonable time within window of specified size, because you only have to replay WAL from lastest base backup.
Barman automatically deletes old backups and WAL according to retetion policy.
Backup source is static — pgmaster node.
In case of master failover, backuping will continue from standby server
Whole backup procedure is performed remotely, but for recovery SSH access is required.

*Before using in production read following documentation:*
 * http://docs.pgbarman.org/release/2.2/index.html
 * https://www.postgresql.org/docs/current/static/continuous-archiving.html

*For Disaster Recovery process see [RECOVERY.md](./doc/RECOVERY.md)*

Barman exposes several metrics on `:8080/metrics` for more information see [Barman docs](./barman/README.md)

## Health-checks

To make sure you cluster works as expected without 'split-brain' or other issues, you have to setup health-checks and stop container if any health-check returns non-zero result. That is really useful when you use Kubernetes which has livenessProbe (check how to use it in [the example](./k8s/example2-single-statefulset/nodes/node.yml)) 

* Postgres containers:
    * `/usr/local/bin/cluster/healthcheck/is_major_master.sh` - detect if node acts as a 'false'-master and there is another master - with more standbys
* Pgpool
    * `/usr/local/bin/pgpool/has_enough_backends.sh [REQUIRED_NUM_OF_BACKENDS, default=$REQUIRE_MIN_BACKENDS]` - check if there are enough backend behind `pgpool`
    * `/usr/local/bin/pgpool/has_write_node.sh` - check if one of the backend can be used as a master with write access



## Useful commands

* Get map of current cluster(on any `postgres` node): 
    * `gosu postgres repmgr cluster show` - tries to connect to all nodes on request ignore status of node in `$(get_repmgr_schema).$REPMGR_NODES_TABLE`
    * `gosu postgres psql $REPLICATION_DB -c "SELECT * FROM $(get_repmgr_schema).$REPMGR_NODES_TABLE"` - just select data from tables
* Get `pgpool` status (on any `pgpool` node): `PGPASSWORD=$CHECK_PASSWORD psql -U $CHECK_USER -h localhost template1 -c "show pool_nodes"`
* In `pgpool` container check if primary node exists: `/usr/local/bin/pgpool/has_write_node.sh` 

Any command might be wrapped with `docker-compose` or `kubectl` - `docker-compose exec {NODE} bash -c '{COMMAND}'` or `kubectl exec {POD_NAME} -- bash -c '{COMMAND}'`


## Scenarios

Check [the document](./doc/FLOWS.md) to understand different cases of failover, split-brain resistance and recovery

## Publications
* [Article on Medium.com](https://medium.com/@dpaunin/postgresql-cluster-into-kubernetes-cluster-f353cde212de)
* [Статья на habr-e](https://habrahabr.ru/post/301370/)

## How to contribute

Check [the doc](./doc/CONTRIBUTE.md) to understand how to contribute

## FAQ

* Example of real/live usage: 
    * [Lazada/Alibaba Group](http://lazada.com/)
* Why not [sorintlab/stolon](https://github.com/sorintlab/stolon):
    * Complex logic with a lot of go-code
    * Non-standard tools for Postgres ecosystem
* [How to promote master, after failover on postgresql with docker](http://stackoverflow.com/questions/37710868/how-to-promote-master-after-failover-on-postgresql-with-docker)
* Killing of node in the middle (e.g. `pgslave1`) will cause [dieing of whole branch](https://groups.google.com/forum/?hl=fil#!topic/repmgr/lPAYlawhL0o)
   * That make seance as second or deeper level of replication should not be able to connect to root master (it makes extra load on server) or change upstream at all


## Documentation and manuals

* Streaming replication in postgres: https://wiki.postgresql.org/wiki/Streaming_Replication
* Repmgr: https://github.com/2ndQuadrant/repmgr
* Pgpool2: http://www.pgpool.net/docs/latest/pgpool-en.html
* Barman: http://www.pgbarman.org/
* Kubernetes: http://kubernetes.io/
