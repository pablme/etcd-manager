# etcd-manager

etcd-manager manages an etcd cluster, on a cloud or on bare-metal.

It borrows from the ideas of the etcd-operator, but avoids the circular dependency of relying on kubernetes (which itself relies on etcd).

The key insight is that we aim to "pivot to etcd" as soon as possible.  So while we use a gossip-like mechanism to discover the initial cluster membership and to elect a coordinator, we then have etcd start a cluster.  Cluster membership changes take place through raft.

## Walkthough

We'll now do a walkthrough to get going with local development.  In production you likely would run this using
a packaged docker container, but this walkthrough lets you see what's going on here.

etcd must be installed in `/opt/etcd-v2.2.1-linux-amd64/etcd`, etcdctl in `/opt/etcd-v2.2.1-linux-amd64/etcdctl`.  Each version of etcd you want to run must be installed in the same pattern.  Make sure you've downloaded `/opt/etcd-v3.2.12-linux-amd64` for this demo. (Typically this will be done in a docker image)

```
bazel build //cmd/etcd-manager
ln -s bazel-bin/cmd/etcd-manager/linux_amd64_stripped/etcd-manager
./etcd-manager --address 127.0.0.1 --members=1 --cluster-name=test --backup-store=file:///tmp/etcd-manager/backups/test --data-dir=/tmp/etcd-manager/data/test/1 --etcd-version=2.2.1 --client-urls=http://127.0.0.1:4001 --quarantine-client-urls=http://127.0.0.1:8001
```

That should start a single node cluster of etcd (`--members=1`), running etcd version 2.2.1 (`--etcd-version=2.2.1`).

You should be able to set and list keys using the etcdctl tool:

```
> curl -XPUT -d "value=world"  http://127.0.0.1:4001/v2/keys/hello
{"action":"set","node":{"key":"/hello","value":"world","modifiedIndex":6,"createdIndex":6}}
> curl http://127.0.0.1:4001/v2/keys/hello
{"action":"get","node":{"key":"/hello","value":"world","modifiedIndex":6,"createdIndex":6}}
```

Now if we want to expand the cluster (it's probably easiest to run each of these commands in different windows / tabs / tmux windows / screen windows):

```
./etcd-manager --address 127.0.0.2 --members=1 --cluster-name=test --backup-store=file:///tmp/etcd-manager/backups/test --data-dir=/tmp/etcd-manager/data/test/2 --client-urls=http://127.0.0.2:4001 --quarantine-client-urls=http://127.0.0.2:8001
./etcd-manager --address 127.0.0.3 --members=1 --cluster-name=test --backup-store=file:///tmp/etcd-manager/backups/test --data-dir=/tmp/etcd-manager/data/test/3 --client-urls=http://127.0.0.3:4001 --quarantine-client-urls=http://127.0.0.3:8001
```

Within a few seconds, the two other nodes will join the gossip cluster, but will not yet be part of etcd.  The leader controller will be logging something like this:

```
etcd cluster state: etcdClusterState
  members:
    {"name":"GEyIUF0s9J07setaePSQ3Q","peerURLs":["http://127.0.0.1:2380"],"clientURLs":["http://127.0.0.1:4001"],"ID":"4be33a4562894143"}
  peers:
    etcdClusterPeerInfo{peer=peer{id:"yFQlUrhyV2P0i14oXydBbw" addresses:"127.0.0.3:8000" }, etcState=cluster_name:"test" node_configuration:<name:"yFQlUrhyV2P0i14oXydBbw" peer_urls:"http://127.0.0.3:2380" client_urls:"http://127.0.0.3:4001" quarantined_client_urls:"http://127.0.0.3:8001" > }
    etcdClusterPeerInfo{peer=peer{id:"GEyIUF0s9J07setaePSQ3Q" addresses:"127.0.0.1:8000" }, etcState=cluster_name:"test" node_configuration:<name:"GEyIUF0s9J07setaePSQ3Q" peer_urls:"http://127.0.0.1:2380" client_urls:"http://127.0.0.1:4001" quarantined_client_urls:"http://127.0.0.1:8001" > etcd_state:<cluster:<cluster_token:"XypoARu3zvWjVHHmWSGr_Q" nodes:<name:"GEyIUF0s9J07setaePSQ3Q" peer_urls:"http://127.0.0.1:2380" client_urls:"http://127.0.0.1:4001" quarantined_client_urls:"http://127.0.0.1:8001" > > etcd_version:"2.2.1" > }
    etcdClusterPeerInfo{peer=peer{id:"mW4JV4NzL-hO39BEU8gnhA" addresses:"127.0.0.2:8000" }, etcState=cluster_name:"test" node_configuration:<name:"mW4JV4NzL-hO39BEU8gnhA" peer_urls:"http://127.0.0.2:2380" client_urls:"http://127.0.0.2:4001" quarantined_client_urls:"http://127.0.0.2:8001" > }
```

Maintaining this state is essentially the core controller loop.  The leader pings each peer,
asking if it is configured to run etcd, and also queries the state of etcd and the desired spec.
Where there are differences, the controller attempts to converge the state by adding/removing members,
doing backups/restores and changing versions.

We can reconfigure the cluster:

```
> curl http://127.0.0.1:4001/v2/keys/kope.io/etcd-manager/test/spec
{"action":"get","node":{"key":"/kope.io/etcd-manager/test/spec","value":"{\n  \"memberCount\": 1,\n  \"etcdVersion\": \"2.2.1\"\n}","modifiedIndex":4,"createdIndex":4}}

> curl -XPUT -d 'value={ "memberCount": 3, "etcdVersion": "2.2.1" }' http://127.0.0.1:4001/v2/keys/kope.io/etcd-manager/test/spec
```

Within a minute, we should see all 3 nodes in the etcd cluster:

```
> curl http://127.0.0.1:4001/v2/members/
{"members":[{"id":"c05691f8951bfaf5","name":"FD2NN12mS_K4ovji8dTL9g","peerURLs":["http://127.0.0.1:2380"],"clientURLs":["http://127.0.0.1:4001"]},{"id":"c3dd045ca41acb5f","name":"ykjVfcnf5DWUCwpKSBQFEg","peerURLs":["http://127.0.0.3:2380"],"clientURLs":["http://127.0.0.3:4001"]},{"id":"d0c3c4397fa244d1","name":"d7qDvoAwX5bhhUwQo0CHgg","peerURLs":["http://127.0.0.2:2380"],"clientURLs":["http://127.0.0.2:4001"]}]}
```

Play around with stopping & starting the nodes, or with removing the data for one at a time.  The cluster should be available whenever a majority of the processes are running,
and as long as you allow the cluster to recover before deleting a second data directory, no data should be lost.
(We'll cover "majority" disaster recovery later)

### Disaster recovery

The etcd-manager performs periodic backups.  In the event of a total failure, it will restore automatically
(TODO: we should make this configurable - if a node _could_ recover we likely want this to be manually triggered)

Verify backup/restore works correctly:

```
curl -XPUT -d "value=world"  http://127.0.0.1:4001/v2/keys/hello
```

* Wait for a "took backup" message (TODO: should we be able to force a backup?)
* Stop all 3 processes
* Remove the active data: `rm -rf /tmp/etcd-manager/data/test`
* Restart all 3 processes

Disaster recovery will detect that no etcd nodes are running, will start a cluster on all 3 nodes, and restore the backup.


### Upgrading

```
> curl http://127.0.0.1:4001/v2/keys/kope.io/etcd-manager/test/spec
{"action":"get","node":{"key":"/kope.io/etcd-manager/test/spec","value":"{ \"memberCount\": 3, \"etcdVersion\": \"2.2.1\" }","modifiedIndex":8,"createdIndex":8}}

> curl -XPUT -d 'value={ "memberCount": 3, "etcdVersion": "3.2.12" }' http://127.0.0.1:4001/v2/keys/kope.io/etcd-manager/test/spec
```

Dump keys to be sure that everything copied across:
```
> ETCDCTL_API=3 /opt/etcd-v3.2.12-linux-amd64/etcdctl --endpoints http://127.0.0.1:4001 get "" --prefix
/hello
world
/kope.io/etcd-manager/test/spec
{ "memberCount": 3, "etcdVersion": "3.2.12" }
```

You may note that we did the impossible here - we went straight from etcd 2 to etcd 3 in an HA cluster.  There was some
downtime during the migration, but we performed a logical backup:

* Quarantined each etcd cluster members, so that they cannot be reached by clients.  This prevents the etcd data from changing.
* Performs a backup (using `etcdctl backup ...`)
* Stops all the etcd members in the cluster
* Creates a new cluster running the new version, quarantined
* Restores the backup into the new cluster
* Lifts the quarantine on the new cluster, so that it is reachable

This procedure works between any two versions of etcd (or even with other storage mechanisms).  It has the downside
that the etcd-version of each object is changed, meaning all watches are invalidated.

TODO: We should enable "hot" upgrades where the version change is compatible.  (It's easy, but it's nice to have one code path for now)

If you want to try a downgrade:
`ETCDCTL_API=3 /opt/etcd-v3.2.12-linux-amd64/etcdctl --endpoints http://127.0.0.1:4001  put /kope.io/etcd-manager/test/spec '{ "memberCount": 3, "etcdVersion": "2.3.7" }'`

## Code overview

### etcd-host

* Each node can host etcd
* The state is stored in a local directory; typically this would be a persistent volume
* etcd-manager runs the local etcd process if configured, restarts it after a crash, etc.
* Code is in `pkg/etcd`

### Leader Election Overview

* Each node that can be a master starts a controller loop.
* Each controller gossips with other controllers that it discovers (initially through seeding, then through gossip)
* Seed discovery is pluggable
* A controller may (or may not) be running an etcd process
* The controllers try to elect a "weak leader", by choosing the member with the lowest ID.
* A controller that believes itself to be the leader sends a message to every peer, and any peer can reject the leadership bid (if it knows of a peer with a lower ID)
* Upon leader election, the controller may try to take a shared lock (for additional insurance against multiple leaders)
* Gossip and leader election code is in `pkg/privateapi`

### Leader Control Loop

Once a leader has been determined, it performs this basic loop:

* It queries every peer to see if it is hosting an etcd process
* If no peer is running etcd, it will try to start etcd.  The leader picks an initial membership set (satisfying quorum) and requests those peers to start the cluster.  (The leader may choose itself)
* If there is an etcd cluster but it has fewer members than desired, the leader will try to pick a node to add to the cluster; it communicates with the peer to start the cluster.
* If there is an etcd cluster but it has more members than desired, the leader will try to remove an etcd cluster member.
* If there is an etcd cluster but it has unhealthy members, the leader will try to remove an unhealthy etcd cluster member.
* Periodically a backup of etcd is performed.

