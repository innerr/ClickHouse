Data replication
----------------

ReplicatedMergeTree
~~~~~~~~~~~~~~~~~~~

ReplicatedCollapsingMergeTree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ReplicatedAggregatingMergeTree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ReplicatedSummingMergeTree
~~~~~~~~~~~~~~~~~~~~~~~~~~

Replication is only supported for tables in the MergeTree family. Replication works at the level of an individual table, not the entire server. A server can store both replicated and non-replicated tables at the same time.

INSERT and ALTER are replicated (for more information, see ALTER). Compressed data is replicated, not query texts.
The CREATE, DROP, ATTACH, DETACH, and RENAME queries are not replicated. In other words, they belong to a single server. The CREATE TABLE query creates a new replicatable table on the server where the query is run. If this table already exists on other servers, it adds a new replica. The DROP TABLE query deletes the replica located on the server where the query is run. The RENAME query renames the table on one of the replicas. In other words, replicated tables can have different names on different replicas.

Replication is not related to sharding in any way. Replication works independently on each shard.

Replication is an optional feature. To use replication, set the addresses of the ZooKeeper cluster in the config file. Example:

.. code-block:: xml

  <zookeeper>
      <node index="1">
          <host>example1</host>
          <port>2181</port>
      </node>
      <node index="2">
          <host>example2</host>
          <port>2181</port>
      </node>
      <node index="3">
          <host>example3</host>
          <port>2181</port>
      </node>
  </zookeeper>

**Use ZooKeeper version 3.4.5 or later** For example, the version in the Ubuntu Precise package is too old.

You can specify any existing ZooKeeper cluster - the system will use a directory on it for its own data (the directory is specified when creating a replicatable table).

If ZooKeeper isn't set in the config file, you can't create replicated tables, and any existing replicated tables will be read-only.

ZooKeeper isn't used for SELECT queries. In other words, replication doesn't affect the productivity of SELECT queries - they work just as fast as for non-replicated tables.

For each INSERT query (more precisely, for each inserted block of data; the INSERT query contains a single block, or per block for every max_insert_block_size = 1048576 rows), approximately ten entries are made in ZooKeeper in several transactions. This leads to slightly longer latencies for INSERT compared to non-replicated tables. But if you follow the recommendations to insert data in batches of no more than one INSERT per second, it doesn't create any problems. The entire ClickHouse cluster used for coordinating one ZooKeeper cluster has a total of several hundred INSERTs per second. The throughput on data inserts (the number of rows per second) is just as high as for non-replicated data.

For very large clusters, you can use different ZooKeeper clusters for different shards. However, this hasn't proven necessary on the Yandex.Metrica cluster (approximately 300 servers).

Replication is asynchronous and multi-master. INSERT queries (as well as ALTER) can be sent to any available server. Data is inserted on this server, then sent to the other servers. Because it is asynchronous, recently inserted data appears on the other replicas with some latency. If a part of the replicas is not available, the data on them is written when they become available. If a replica is available, the latency is the amount of time it takes to transfer the block of compressed data over the network.

There are no quorum writes. You can't write data with confirmation that it was received by more than one replica. If you write a batch of data to one replica and the server with this data ceases to exist before the data has time to get to the other replicas, this data will be lost.

Each block of data is written atomically. The INSERT query is divided into blocks up to max_insert_block_size = 1048576 rows. In other words, if the INSERT query has less than 1048576 rows, it is made atomically.

Blocks of data are duplicated. For multiple writes of the same data block (data blocks of the same size containing the same rows in the same order), the block is only written once. The reason for this is in case of network failures when the client application doesn't know if the data was written to the DB, so the INSERT query can simply be repeated. It doesn't matter which replica INSERTs were sent to with identical data - INSERTs are idempotent. This only works for the last 100 blocks inserted in a table.

During replication, only the source data to insert is transferred over the network. Further data transformation (merging) is coordinated and performed on all the replicas in the same way. This minimizes network usage, which means that replication works well when replicas reside in different datacenters. (Note that duplicating data in different datacenters is the main goal of replication.)

You can have any number of replicas of the same data. Yandex.Metrica uses double replication in production. Each server uses RAID-5 or RAID-6, and RAID-10 in some cases. This is a relatively reliable and convenient solution.

The system monitors data synchronicity on replicas and is able to recover after a failure. Failover is automatic (for small differences in data) or semi-automatic (when data differs too much, which may indicate a configuration error).

Creating replicated tables
~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``'Replicated'`` prefix is added to the table engine name. For example, ``ReplicatedMergeTree``.

Two parameters are also added in the beginning of the parameters list - the path to the table in ZooKeeper, and the replica name in ZooKeeper.

Example:
.. code-block:: text

  ReplicatedMergeTree('/clickhouse/tables/{layer}-{shard}/hits', '{replica}', EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID), EventTime), 8192)

As the example shows, these parameters can contain substitutions in curly brackets. The substituted values are taken from the 'macros' section of the config file. Example:

.. code-block:: xml

  <macros>
      <layer>05</layer>
      <shard>02</shard>
      <replica>example05-02-1.yandex.ru</replica>
  </macros>

The path to the table in ZooKeeper should be unique for each replicated table. Tables on different shards should have different paths.
In this case, the path consists of the following parts:

``/clickhouse/tables/`` - is the common prefix. We recommend using exactly this one.

``{layer}-{shard}`` - is the shard identifier. In this example it consists of two parts, since the Yandex.Metrica cluster uses bi-level sharding. For most tasks, you can leave just the {shard} substitution, which will be expanded to the shard identifier.

``hits`` - is the name of the node for the table in ZooKeeper. It is a good idea to make it the same as the table name. It is defined explicitly, because in contrast to the table name, it doesn't change after a RENAME query.

The replica name identifies different replicas of the same table. You can use the server name for this, as in the example. The name only needs to be unique within each shard.

You can define everything explicitly instead of using substitutions. This might be convenient for testing and for configuring small clusters, but it is inconvenient when working with large clusters.

Run CREATE TABLE on each replica. This query creates a new replicated table, or adds a new replica to an existing one.

If you add a new replica after the table already contains some data on other replicas, the data will be copied from the other replicas to the new one after running the query. In other words, the new replica syncs itself with the others.

To delete a replica, run DROP TABLE. However, only one replica is deleted - the one that resides on the server where you run the query.

Recovery after failures
~~~~~~~~~~~~~~~~~~~~~~~

If ZooKeeper is unavailable when a server starts, replicated tables switch to read-only mode. The system periodically attempts to connect to ZooKeeper.

If ZooKeeper is unavailable during an INSERT, or an error occurs when interacting with ZooKeeper, an exception is thrown.

After connecting to ZooKeeper, the system checks whether the set of data in the local file system matches the expected set of data (ZooKeeper stores this information). If there are minor inconsistencies, the system resolves them by syncing data with the replicas.

If the system detects broken data parts (with the wrong size of files) or unrecognized parts (parts written to the file system but not recorded in ZooKeeper), it moves them to the 'detached' subdirectory (they are not deleted). Any missing parts are copied from the replicas.

Note that ClickHouse does not perform any destructive actions such as automatically deleting a large amount of data.

When the server starts (or establishes a new session with ZooKeeper), it only checks the quantity and sizes of all files. If the file sizes match but bytes have been changed somewhere in the middle, this is not detected immediately, but only when attempting to read the data for a SELECT query. The query throws an exception about a non-matching checksum or size of a compressed block. In this case, data parts are added to the verification queue and copied from the replicas if necessary.

If the local set of data differs too much from the expected one, a safety mechanism is triggered. The server enters this in the log and refuses to launch. The reason for this is that this case may indicate a configuration error, such as if a replica on a shard was accidentally configured like a replica on a different shard. However, the thresholds for this mechanism are set fairly low, and this situation might occur during normal failure recovery. In this case, data is restored semi-automatically - by "pushing a button".

To start recovery, create the node ``/path_to_table/replica_name/flags/force_restore_data`` in ZooKeeper with any content or run command to recover all replicated tables:

.. code-block:: text

  sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data

Then launch the server. On start, the server deletes these flags and starts recovery.

Recovery after complete data loss
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If all data and metadata disappeared from one of the servers, follow these steps for recovery:

#. Install ClickHouse on the server. Define substitutions correctly in the config file that contains the shard identifier and replicas, if you use them.
#. If you had unreplicated tables that must be manually duplicated on the servers, copy their data from a replica (in the directory /var/lib/clickhouse/data/db_name/table_name/).
#. Copy table definitions located in /var/lib/clickhouse/metadata/. from a replica. If a shard or replica identifier is defined explicitly in the table definitions, correct it so that it corresponds to this replica. (Alternatively, launch the server and make all the ATTACH TABLE queries that should have been in the .sql files in /var/lib/clickhouse/metadata/.)
#. Create the ``/path_to_table/replica_name/flags/force_restore_data`` node in ZooKeeper with any content or run command to recover all replicated tables: ``sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data``

Then launch the server (restart it if it is already running). Data will be downloaded from replicas.

An alternative recovery option is to delete information about the lost replica from ZooKeeper ( ``/path_to_table/replica_name``), then create the replica again as described in "Creating replicated tables".

There is no restriction on network bandwidth during recovery. Keep this in mind if you are restoring many replicas at once.

Converting from MergeTree to ReplicatedMergeTree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From here on, we use ``MergeTree`` to refer to all the table engines in the ``MergeTree`` family, including ``ReplicatedMergeTree``.

If you had a MergeTree table that was manually replicated, you can convert it to a replicatable table. You might need to do this if you have already collected a large amount of data in a MergeTree table and now you want to enable replication.

If the data differs on various replicas, first sync it, or delete this data on all the replicas except one.

Rename the existing MergeTree table, then create a ReplicatedMergeTree table with the old name.
Move the data from the old table to the 'detached' subdirectory inside the directory with the new table data (``/var/lib/clickhouse/data/db_name/table_name/``).
Then run ALTER TABLE ATTACH PART on one of the replicas to add these data parts to the working set.

If exactly the same parts exist on the other replicas, they are added to the working set on them. If not, the parts are downloaded from the replica that has them.

Converting from ReplicatedMergeTree to MergeTree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a MergeTree table with a different name. Move all the data from the directory with the ReplicatedMergeTree table data to the new table's data directory. Then delete the ReplicatedMergeTree table and restart the server.

If you want to get rid of a ReplicatedMergeTree table without launching the server:
 * Delete the corresponding .sql file in the metadata directory (``/var/lib/clickhouse/metadata/``).
 * Delete the corresponding path in ZooKeeper (``/path_to_table/replica_name``).
After this, you can launch the server, create a MergeTree table, move the data to its directory, and then restart the server.

Recovery when metadata in the ZooKeeper cluster is lost or damaged
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you lost ZooKeeper, you can save data by moving it to an unreplicated table as described above.
