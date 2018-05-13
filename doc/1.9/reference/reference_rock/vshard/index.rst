.. _vshard-module:

================================================================================
Module `vshard`
================================================================================
-------------------------------------------------------------------------------
Overview
-------------------------------------------------------------------------------

In Tarantool, a horizontal scaling is performed via sharding. Sharding is introduced
by a ``vshard`` module. With sharding, the tuples of a dataset are distributed across
multiple nodes, with a Tarantool database server instance on each node. Each instance
handles only a subset of the total data, so larger loads can be handled by simply
adding more servers.

The ``vshard`` module is based on the concept of virtual buckets, where a tuple
set is partitioned into a large number of abstract virtual nodes (virtual buckets,
or buckets) rather than into a smaller number of physical nodes.

Hashing a sharding key into a large number of virtual buckets allows seamlessly
changing the number of servers in the cluster. The rebalancing mechanism distributes
buckets evenly among all shards in case some servers were added or removed.

The buckets have states, so it is easy to monitor the server states. For example,
a server instance is active and available for all types of requests, or a failover
occurred and the instance accepts only read requests.

The ``vshard`` module provides analogs for the data-manipulation functions of the
Tarantool box library (select, insert, replace, update, delete).

-------------------------------------------------------------------------------
Installation
-------------------------------------------------------------------------------

The ``vshard`` package is distributed separately from the main Tarantool package.
To acquire it, do a separate installation:

.. code-block:: console

    tarantoolctl rocks install https://raw.githubusercontent.com/tarantool/vshard/master/vshard-scm-1.rockspec

.. NOTE::

    The ``vshard`` package requires Tarantool version 1.9+.

-------------------------------------------------------------------------------
Quick start
-------------------------------------------------------------------------------

The ``vshard/example/`` directory includes a pre-configured development cluster
of 1 ``router`` and 2 replica sets of 2 nodes (2 ``storages``) each, making 5
Tarantool instances in total:

* ``router_1`` – a ``router`` instance
* ``storage_1_a`` – a ``storage`` instance, the master of the first replica set
* ``storage_1_b`` – a ``storage`` instance, the replica of the first replica set
* ``storage_2_a`` – a ``storage`` instance, the master of the second replica set
* ``storage_2_b`` – a ``storage`` instance, the replica of the second replica set

All instances are managed using the ``tarantoolctl`` utility from the root directory
of the project.

Change the directory to ``example/`` and use make to run the development cluster:

.. code-block:: console

    # cd example/
    # make
    tarantoolctl stop storage_1_a  # stop the first storage instance
    Stopping instance storage_1_a...
    tarantoolctl stop storage_1_b
    <...>
    rm -rf data/
    tarantoolctl start storage_1_a # start the first storage instance
    Starting instance storage_1_a...
    Starting configuration of replica 8a274925-a26d-47fc-9e1b-af88ce939412
    I am master
    Taking on replicaset master role...
    Run console at unix/:./data/storage_1_a.control
    started
    mkdir ./data/storage_1_a
    <...>
    tarantoolctl start router_1 # start the router
    Starting instance router_1...
    Starting router configuration
    Calling box.cfg()...
    <...>
    Run console at unix/:./data/router_1.control
    started
    mkdir ./data/router_1
    Waiting cluster to start
    echo "vshard.router.bootstrap()" | tarantoolctl enter router_1
    connected to unix/:./data/router_1.control
    unix/:./data/router_1.control> vshard.router.bootstrap()
    ---
    - true
    ...
    unix/:./data/router_1.control>
    tarantoolctl enter router_1 # enter the admin console
    connected to unix/:./data/router_1.control
    unix/:./data/router_1.control>

Some ``tarantoolctl`` commands:

* ``tarantoolctl start router_1`` – start the router instance
* ``tarantoolctl enter router_1``  – enter the admin console

.. Add link

The full list of tarantoolctl commands for managing Tarantool instances is available
in the Commands for managing Tarantool instances section of the Tarantool manual.

Essential make commands you need to know:

* ``make start`` – start all Tarantool instances
* ``make stop`` – stop all Tarantool instances
* ``make logcat`` – show logs from all instances
* ``make enter`` – enter the admin console on router_1
* ``make clean`` – clean up all persistent data
* ``make test`` – run the test suite (you can also run test-run.py in the test directory)
* ``make`` – execute ``make stop``, ``make clean``, ``make start`` and ``make enter``

For example, to start all instances, use ``make start``:

.. code-block:: console

    # make start
    # ps x|grep tarantool
    46564   ??  Ss     0:00.34 tarantool storage_1_a.lua <running>
    46566   ??  Ss     0:00.19 tarantool storage_1_b.lua <running>
    46568   ??  Ss     0:00.35 tarantool storage_2_a.lua <running>
    46570   ??  Ss     0:00.20 tarantool storage_2_b.lua <running>
    46572   ??  Ss     0:00.25 tarantool router_1.lua <running>

To perform commands in the admin console, use the ``router`` API:

.. code-block:: console

    unix/:./data/router_1.control> vshard.router.info()
    ---
    - replicasets:
        ac522f65-aa94-4134-9f64-51ee384f1a54:
          replica: &0
            network_timeout: 0.5
            status: available
            uri: storage@127.0.0.1:3303
            uuid: 1e02ae8a-afc0-4e91-ba34-843a356b8ed7
          uuid: ac522f65-aa94-4134-9f64-51ee384f1a54
          master: *0
        cbf06940-0790-498b-948d-042b62cf3d29:
          replica: &1
            network_timeout: 0.5
            status: available
            uri: storage@127.0.0.1:3301
            uuid: 8a274925-a26d-47fc-9e1b-af88ce939412
          uuid: cbf06940-0790-498b-948d-042b62cf3d29
          master: *1
      bucket:
        unreachable: 0
        available_ro: 0
        unknown: 0
        available_rw: 3000
      status: 0
      alerts: []
    ...

-------------------------------------------------------------------------------
About sharding
-------------------------------------------------------------------------------
*******************************************************************************
Why sharding
*******************************************************************************

Sharding is a database architecture that allows distributing a dataset across
multiple servers. An initial dataset is partitioned into multiple parts, so each
part is stored on a separate server. The dataset is partitioned using shard keys.
While a project is growing, scaling of the databases may become the most challenging
issue. Once a single server cannot withstand the load, scaling methods should be
applied. There are two different approaches for scaling data: vertical and horizontal
scaling.

*Vertical scaling* implies that the hardware capacities of a single server would
be increased.

*Horizontal scaling* implies that a dataset is partitioned and distributed over
multiple servers. In case new servers are added, the dataset is re-distributed
evenly across all servers, both the original and new ones.

Tarantool supports horizontal scaling through sharding.

*******************************************************************************
Sharded cluster components
*******************************************************************************

A sharded cluster in Tarantool consists of storages, routers, and a rebalancer.

A **storage** is a node storing a subset of a dataset. Multiple replicated storages
are deployed as replica sets to provide redundancy (a replica set can also be
referred as shard).

A **router** is a standalone software component that routes read and write requests
from the client application to shards.

A **rebalancer** is an internal component that distributes the dataset among all
shards evenly in case some servers are added or removed. It also balances the load
considering the capacities of existing replica sets.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Storage
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Storage** is a node storing a subset of a dataset. Multiple replicated storages
comprise a replica set. Each storage in a replica set has a role, **master** or
**replica**. Master processes read and write requests. Replicas process read
requests, but cannot process write requests.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Virtual buckets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The sharded dataset is partitioned into a large number of abstract nodes called
**virtual buckets** (further referred as **buckets**).

The dataset is partitioned using the shard key (or **bucket id**, in terms of
Tarantool). Bucket id is a number from 1 to N, where N is the total number of
buckets.

Each replica set stores a unique subset of buckets. One bucket cannot belong to
multiple replica sets at a time.

The total number of buckets is determined by the administrator who sets up the
initial cluster configuration.

Every Tarantool space you plan to shard must have a bucket id field indexed by the
bucket id ``index``. Spaces without the bucket id indexes don’t participate in sharding
but can be used as regular spaces. By default, the name of the index coincides with
the bucket id.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Migration of buckets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A **rebalancer** is a background rebalancing process that ensures an even
distribution of buckets across the shards. During rebalancing, buckets are being
migrated among replica sets.

A replica set from which the bucket is being migrated is called **source**; a
target replica set to which the bucket is being migrated is called **destination**.

A **replica set lock** makes a replica set invisible to the rebalancer. A locked
replica set can neither receive new buckets nor migrate its own ones.

While being migrated, the bucket can have different states:

* ACTIVE – the bucket is available for read and write requests.
* PINNED – the bucket is locked for migrating to another replica set. Otherwise
  pinned buckets are similar to the buckets in the ACTIVE state.
* SENDING – the bucket is currently being copied to the destination replica set
  read requests to the source replica set are still processed.
* RECEIVING – the bucket is currently being filled; all requests to it are rejected.
* SENT – the bucket was migrated to the destination replica set.
* GARBAGE – the bucket was already migrated to the destination replica set during
  rebalancing; or the bucket was initially in the RECEIVING state, but some error
  occurred during the migration.

Buckets in the GARBAGE state are deleted by the garbage collector.

Altogether, migration is performed as follows:

1. At the destination replica set, a new bucket is created and assigned the RECEIVING
   state, the data copying starts, and the bucket rejects all requests.
2. The source bucket at the source replica set is assigned the SENDING state, and
   the bucket continues to process read requests.
3. Once the data is copied, the bucket on the source replica set is marked SENT and it starts rejecting all requests.
4. The bucket on the destination replica set goes into the ACTIVE state and starts
   accepting all requests.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The `_bucket` system space
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The “**_bucket**” system space of each replica set stores the ids of buckets present
in the replica set. The space contains the following tuples:

* ``bucket`` – bucket id
* ``status`` – state of the bucket
* ``destination`` – uuid of the destination replica set

An example of ``_bucket.select{}``:

.. code-block:: tarantoolsession

    ---
    - - [1, ACTIVE, abfe2ef6-9d11-4756-b668-7f5bc5108e2a]
      - [2, SENT, 19f83dcb-9a01-45bc-a0cf-b0c5060ff82c]
    ...

Once the bucket is migrated, the destination replica set uuid is filled in the
table. While the bucket is still located on the source replica set, the value of
the destination replica set uuid is equal to ``NULL``.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Router
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All requests from the application come to the sharded cluster through a ``router``.
The ``router`` keeps the topology of a sharded cluster transparent for the application,
thus keeping the application unaware of:

* the number and location of shards,
* data rebalancing process,
* the fact and the process of a failover that occurred after a replica's failure.

The router does not have a persistent state, nor does it store the cluster topology
or balance the data. The router is a standalone software component that can run
in the storage layer or application layer depending on the application features.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Routing table
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

А routing table on the ``router`` stores the map of all bucket ids to replica sets.
It ensures the consistency of sharding in case of failover.

The ``router`` keeps a persistent pool of connections to all the storages that
are created at startup. This helps prevent configuration errors. Once the connection
pool is created, the router caches the current state of the routing table in order
to speed up routing. If a bucket migrated to another ``storage`` after rebalancing,
or a failover occurred and caused one of the shards switching to another replica,
the ``discovery fiber`` on the router updates the routing table automatically.

As the bucket id is explicitly indicated both in the data and in the mapping table
on the router, the data is consistent regardless of the application logic. It also
makes rebalancing transparent for the application.

-------------------------------------------------------------------------------
Processing requests
-------------------------------------------------------------------------------

Requests to the database can be performed by the application or using stored
procedures. Either way, the bucket id should be explicitly specified in the request.

All requests are forwarded to the ``router`` first. The only operation supported
by the router is ``call``. The operation is performed via ``vshard.router.call()``
function:

``result = vshard.router.call(<bucket_id>, <mode(read:write)>, <function_name>, {<argument_list>}, {<opts>})``

Requests are processed as follows:

1. The ``router`` uses the bucket id to search for a replica set with the
   corresponding bucket in the routing table.

   If the map of the bucket id to the replica set is not known to the ``router``
   (the discovery fiber hasn’t filled the table yet), the router makes requests
   to all ``storages`` to find out where the bucket is located.
2. Once the bucket is located, the shard checks:

   * whether the bucket is stored in the ``_bucket`` system space of the replica set
   * whether the bucket is ACTIVE or PINNED (for a read request, it can also be SENDING)
3. If all the checks succeed, the request is executed. Otherwise, it is terminated
   with the error: ``“wrong bucket”``.

-------------------------------------------------------------------------------
Administration
-------------------------------------------------------------------------------
*******************************************************************************
Configuring a sharded cluster
*******************************************************************************

A minimal viable sharded cluster should consist of:

* one or more replica sets with two or more ``storage`` instances in each
* one or more ``router`` instances

The number of ``storage`` instances in a replica set defines the redundancy factor
of the data. The recommended value is 3 or more. The number of the router instances
is not limited, because routers are completely stateless. We recommend increasing
the number of routers when the existing ``router`` instance becomes CPU or I/O bound.

As the router and ``storage`` applications perform completely different sets of functions,
they should be deployed to different Tarantool instances. Although it is technically
possible to place the router application to every ``storage`` node, this approach is
highly discouraged and should be avoided on production deployments.

All ``storage`` instances can be deployed using identical instance (configuration)
files.

Self-identification is currently performed using ``tarantoolctl``:

.. code-block:: console

    tarantoolctl instance_name

All ``router`` instances can also be deployed using identical instance (configuration)
files.

All cluster nodes must share a common topology. You as an administrator must
ensure that the configurations are identical. We suggest using a configuration
management tool like Ansible or Puppet to deploy the cluster.

Sharding is not integrated into any system for centralized configuration management.
It is implied that the application itself is responsible for interacting with such
a system and passing the sharding parameters.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Sample configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The configuration of a simple sharded cluster can look like this:

.. code-block:: kconfig

    local cfg = {
        memtx_memory = 100 * 1024 * 1024,
        replication_connect_quorum = 0,
        bucket_count = 10000,
        rebalancer_disbalance_threshold = 10,
        rebalancer_max_receiving = 100,
        sharding = {
            ['cbf06940-0790-498b-948d-042b62cf3d29'] = {
                replicas = {
                    ['8a274925-a26d-47fc-9e1b-af88ce939412'] = {
                        uri = 'storage:storage@127.0.0.1:3301',
                        name = 'storage_1_a',
                        master = true
                    },
                    ['3de2e3e1-9ebe-4d0d-abb1-26d301b84633'] = {
                        uri = 'storage:storage@127.0.0.1:3302',
                        name = 'storage_1_b'
                    }
                },
            },
            ['ac522f65-aa94-4134-9f64-51ee384f1a54'] = {
                replicas = {
                    ['1e02ae8a-afc0-4e91-ba34-843a356b8ed7'] = {
                        uri = 'storage:storage@127.0.0.1:3303',
                        name = 'storage_2_a',
                        master = true
                    },
                    ['001688c3-66f8-4a31-8e19-036c17d489c2'] = {
                        uri = 'storage:storage@127.0.0.1:3304',
                        name = 'storage_2_b'
                    }
                },
            },
        },
    }

This cluster includes one ``router`` instance and two ``storage`` instances.
Each ``storage`` instance includes one master and one replica.

The sharding field defines the logical topology of a sharded Tarantool cluster.
All the other fields are passed to ``box.cfg()`` as they are, without modifications.
See the Configuration reference section for details.

On routers call ``vshard.router.cfg(cfg)``:

.. code-block:: console

    cfg.listen = 3300

    -- Start the database with sharding
    vshard = require('vshard')
    vshard.router.cfg(cfg)

On storages call ``vshard.storage.cfg(cfg, instance_uuid)``:

.. code-block:: console

    -- Get instance name
    local MY_UUID = "de0ea826-e71d-4a82-bbf3-b04a6413e417"

    -- Call a configuration provider
    local cfg = require('localcfg')

    -- Start the database with sharding
    vshard = require('vshard')
    vshard.storage.cfg(cfg, MY_UUID)


``vshard.storage.cfg()`` automatically calls box.cfg()and configures the listen
port and replication parameters.

See ``router.lua`` and ``storage.lua`` in the ``vshard/example`` directory for
a sample configuration.

*******************************************************************************
Replica weights
*******************************************************************************

The ``router`` sends all requests to the master instance only. Setting replica
weights allows sending read requests not only to the master instance, but to any
available replica that is the 'nearest'  to the router. Weights are used to define
distances between replicas within a replica set.

Weights can be used, for example, to define the physical distance between the
``router`` and each replica in each replica set. In such a case read requests
are sent to the literary nearest replica.

Setting weights can also help to define the most powerful replicas: the ones that
can process the largest number requests per second.

The idea is to specify the zone for every ``router`` and every replica, therefore
filling a matrix of relative zone weights. This approach allows setting different
weights in different zones for the same replica set.

To set weights, use the zone attribute for each replica in configuration:

.. code-block:: kconfig

    local cfg = {
       sharding = {
          ['...replicaset_uuid...'] = {
             replicas = {
                ['...replica_uuid...'] = {
                     ...,
                     zone = <number or string>
                }
             }
          }
       }
    }

Then, specify relative weights for each zone pair in the weights parameter of
``vshard.router.cfg``. For example:

.. code-block:: kconfig

    weights = {
        [1] = {
            [2] = 1, -- routers of the 1st zone see the weight of the 2nd zone as 1
            [3] = 2, -- routers of the 1st zone see the weight of the 3rd zone as 2


       [4] = 3, -- ...
        },
        [2] = {
            [1] = 10,
            [2] = 0,
            [3] = 10,
            [4] = 20,
        },
        [3] = {
            [1] = 100,
            [2] = 200, -- routers of the 3rd zone see the weight of the 2nd zone as 200. Mind that it is not equal to the weight of the 2nd zone = 2 visible from the 1st zone
            [4] = 1000,
        }
    }

    local cfg = vshard.router.cfg({weights = weights, sharding = ...})

*******************************************************************************
Replica set weights
*******************************************************************************

A replica set weight is not the same as the replica weight. The weight of a replica
set defines the capacity of the replica set: the larger the weight, the more
buckets the replica set can store. The total size of all sharded spaces in the
replica set is also its capacity metric.

You can consider replica set weights as the relative amount of data within a
replica set. For example, if ``replicaset_1 = 100``, and ``replicaset_2 = 200``,
the second replica set stores twice as many buckets as the first one. By default,
all weights of all replica sets are equal.

You can use weights, for example, to store the prevailing amount of data on a
replica set with more memory space.

*******************************************************************************
Rebalancing process
*******************************************************************************

There is an **etalon number** of buckets per a replica set. If there is no deviation
from this number on all the replica set, then the buckets are distributed evenly.

The etalon number is calculated automatically considering the number of buckets
in the cluster and weights of the replica sets.

For example: The user specified the number of buckets equal to 3000, and weights
of 3 replica sets equal to 1, 0.5, and 1.5. The resulting etalon numbers of buckets
for the replica sets are: 1st replica set – 1000, 2nd replica set – 500, 3rd
replica set – 1500.

This approach allows assigning a zero weight to a replica set, which initiates
migration of its buckets to the remaining cluster nodes. It also allows adding
a new zero-load replica set, which initiates migration of the buckets from the
loaded replica sets to the zero-load replica set.

.. NOTE::

    A new zero-load replica set should be assigned a weight for rebalancing to start.

The ``rebalancer`` wakes up periodically and redistributes data from the most
loaded nodes to less loaded nodes. Rebalancing starts if the disbalance threshold
of a replica set exceeds a disbalance threshold specified in the configuration.

The disbalance threshold is calculated as follows:

.. code-block:: none

    |etalon_bucket_number - real_bucket_number| / etalon_bucket_number * 100

When a new shard is added, the configuration can be updated dynamically:

1. The configuration should be updated on all the routers first, and then on all
   the ``storages``.
2. The new shard becomes available for rebalancing in the ``storage`` layer.
3. As a result of rebalancing, buckets are being migrated to the new shard.
4. If a migrated bucket is requested, ``router`` receives an error code containing
   information about the new location of the bucket.

At this time, the new shard is already present in the ``router``'s pool of
connections, so redirection is transparent for the application.

*******************************************************************************
Replica set lock and bucket pin
*******************************************************************************

A replica set lock makes a replica set invisible for the ``rebalancer``: a locked
replica set can neither receive new buckets, nor migrate its own ones.

A bucket pin blocks a specific bucket from migrating: a pinned bucket stays on
the replica set to which it is pinned, until unpinned.

Pinning all replica set buckets is not equal to locking a replica set. Even if
you pin all buckets, a non-locked replica set can still receive new buckets.

Replica set lock is helpful, for example, to separate a replica set from production
replica sets for testing, or to preserve some application metadata that must not
be sharded for a while. A bucket pin is used for similar cases but in a smaller scope.

Introducing both locking a replica set and pinning all buckets is done for the
ability to isolate an entire replica set.

Locked replica sets and pinned buckets affect the rebalancing algorithm as the
``rebalancer`` must ignore locked replica sets and consider pinned buckets when
attempting to reach the best possible balance.

The issue is not trivial as a user can pin too many buckets to a replica set,
so a perfect balance becomes unreachable. For example, look at the following
cluster (assume all replica set weights are equal to 1).

The initial configuration:

.. code-block:: none

    rs1: bucket_count = 150
    rs2: bucket_count = 150, pinned_count = 120

Adding a new replica set:

.. code-block:: none

    rs1: bucket_count = 150
    rs2: bucket_count = 150, pinned_count = 120
    rs3: bucket_count = 0

The perfect balance would be ``100 - 100 - 100``, which is impossible since the
``rs2`` replica set has 120 pinned buckets. The best possible balance here is the
following:

.. code-block:: none

    rs1: bucket_count = 90
    rs2: bucket_count = 120, pinned_count 120
    rs3: bucket_count = 90

The ``rebalancer`` moved as many buckets as possible from ``rs2`` to decrease the
disbalance. At the same time it respected equal weights of ``rs1`` and ``rs3``.

The algorithms of considering locks and pins are completely different, although
they look similar in terms of functionality.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Replica set lock and rebalancing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Locked replica sets simply don’t participate in rebalancing. This means that
even if the actual total number of buckets is not equal to the etalon number,
the disbalance cannot be fixed due to lock. When the rebalancer detects that
one of the replica sets is locked, it recalculates the etalon number of buckets
of the non-locked replica sets as if the locked replica set and its buckets didn’t
exist at all.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Bucket pin and rebalancing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Rebalancing replica sets with pinned buckets requires a more complex algorithm.
Here pinned_count[o] is the number of pinned buckets, and ``etalon_count`` is
the etalon number of buckets per a replica set:

1. The ``rebalancer`` calculates the etalon number of buckets as if all buckets
   were not pinned. Then the rebalancer checks each replica set and compares the
   etalon number of buckets with the number of pinned buckets on a replica set.
   If ``pinned_count < etalon_count``, non-locked replica sets (on this step all
   locked replica sets already are filtered out) with pinned buckets can receive
   new buckets.
2. If ``pinned_count > etalon_count``, the disbalance cannot be fixed, as the
   ``rebalancer`` cannot move pinned buckets out of this replica set. In such a case
   the etalon number is updated and set equal to the number of pinned buckets.
   The replica sets with ``pinned_count > etalon_count`` are not processed by
   the ``rebalancer``, and the number of pinned buckets is subtracted from the
   total number of buckets. The rebalancer tries to move out as many buckets as
   possible from such replica sets.
3. The described procedure is restarted from the step 1 for replica sets with
   ``pinned_count >= etalon_count`` until ``pinned_count <= etalon_count`` on
   all replica sets. The procedure is also restarted when the total number of
   buckets is changed.

The pseudocode for the algorithm is the following:

.. code-block:: console

    function cluster_calculate_perfect_balance(replicasets, bucket_count)
            -- rebalance the buckets using weights of the still viable replica sets --
    end;

    cluster = <all of the non-locked replica sets>;
    bucket_count = <the total number of buckets in the cluster>;
    can_reach_balance = false
    while not can_reach_balance do
            can_reach_balance = true
            cluster_calculate_perfect_balance(cluster, bucket_count);
            foreach replicaset in cluster do
                    if replicaset.perfect_bucket_count <
                       replicaset.pinned_bucket_count then
                            can_reach_balance = false
                            bucket_count -= replicaset.pinned_bucket_count;
                            replicaset.perfect_bucket_count =
                                    replicaset.pinned_bucket_count;
                    end;
            end;
    end;
    cluster_calculate_perfect_balance(cluster, bucket_count);

The complexity of the algorithm is ``O(N^2)``, where N is the number of replica sets.
On each step, the algorithm either finishes the calculation, or ignores at least
one new replica set overloaded with the pinned buckets, and updates the etalon
number of buckets on other replica sets.

*******************************************************************************
Defining spaces
*******************************************************************************

Spaces should be defined within a storage application using ``box.once()``.
For example:

.. code-block:: lua

    box.once("testapp:schema:1", function()
        local customer = box.schema.space.create('customer')
        customer:format({
            {'customer_id', 'unsigned'},
            {'bucket_id', 'unsigned'},
            {'name', 'string'},
        })
        customer:create_index('customer_id', {parts = {'customer_id'}})
        customer:create_index('bucket_id', {parts = {'bucket_id'}, unique = false})

        local account = box.schema.space.create('account')
        account:format({
            {'account_id', 'unsigned'},
            {'customer_id', 'unsigned'},
            {'bucket_id', 'unsigned'},
            {'balance', 'unsigned'},
            {'name', 'string'},
        })
        account:create_index('account_id', {parts = {'account_id'}})
        account:create_index('customer_id', {parts = {'customer_id'}, unique = false})
        account:create_index('bucket_id', {parts = {'bucket_id'}, unique = false})
        box.snapshot()

        box.schema.func.create('customer_lookup')
        box.schema.role.grant('public', 'execute', 'function', 'customer_lookup')
        box.schema.func.create('customer_add')
    end)

*******************************************************************************
Bootstrapping and restarting a storage
*******************************************************************************

If a replica set master fails, it is recommended to:

1. Switch one of the replicas into the master mode. It allows the new master
   to process all the incoming requests.
2. Update the configuration of all the cluster members. It forwards all the
   requests to the new master.

Monitoring the master and switching the instance modes can be handled by any
external utility.

To perform a scheduled downtime of a replica set master, it is recommended to:

1. Update the configuration of the master and wait for the replicas to get into
   sync. All the requests then are forwarded to a new master.
2. Switch another instance into the master mode.
3. Update the configuration of all the nodes.
4. Shut down the old master.

To perform a scheduled downtime of a replica set, it is recommended to:

1. Migrate all the buckets to the other cluster storages.
2. Update the configuration of all the nodes.
3. Shut down the replica set.

In case a whole replica set fails, some part of the dataset becomes inaccessible.
Meanwhile, the ``router`` tries to reconnect to the master of the failed replica
set. This way, once the replica set is up and running again, the cluster is
automatically restored.

*******************************************************************************
Fibers
*******************************************************************************

Search for buckets, buckets recovery, and buckets rebalancing are performed
automatically and do not require human intervention.

Technically, there are multiple fibers responsible for different types of
operations:

* a **discovery** fiber on the ``router`` searches for buckets in the background
* a **failover** fiber on the ``router`` maintains replica connections
* a **garbage** collector fiber on each master ``storage`` removes the contents
  of buckets that were moved
* a **bucket** recovery fiber on each master ``storage`` recovers buckets in the
  SENDING and RECEIVING states in case of reboot
* a **rebalancer** on a single master ``storage`` among all replica sets executes
  the rebalancing process.

  See the Rebalancing process section for details.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Garbage collector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A **garbage collection** fiber is running in the background on the master storages
of each replica set. It starts deleting the contents of the bucket in the GARBAGE
state part by part. Once the bucket is empty, its record is deleted from the
``_bucket`` system space.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Bucket recovery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A **bucket recovery** fiber is running on the master storages. It helps to recover
buckets in the SENDING and RECEIVING states in case of reboot.

Buckets in the SENDING state are recovered as follows:

1. The system first searches for buckets in the SENDING state.
2. If such a bucket is found, the system sends a request to the destination
   replica set.
3. If the bucket on the destination replica set is ACTIVE, the original bucket
   is deleted from the source node.

Buckets in the RECEIVING state are deleted without extra checks.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Failover
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A failover fiber is running on every router. If a master of a replica set
becomes unavailable, the failover redirects read requests to the replicas.
Write requests are rejected with an error until the master becomes available.

-------------------------------------------------------------------------------
Configuration reference
-------------------------------------------------------------------------------
*******************************************************************************
Basic parameters
*******************************************************************************

.. include:: cfg_basic.rst

*******************************************************************************
Replica set functions
*******************************************************************************

.. include:: cfg_replica_set.rst

-------------------------------------------------------------------------------
API reference
-------------------------------------------------------------------------------
*******************************************************************************
Router API
*******************************************************************************

.. include:: router_api.rst
