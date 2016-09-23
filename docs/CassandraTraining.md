# Cassandra Training

* Replication Factor (RF) - typically production implementation is 3.  A single piece of data is replicated to three nodes.
* Consistency Level (ALL, QUORUM, ONE) - how many nodes need to acknowledge data was received before notifying client that the data was written successfully.  Replication to other nodes is still done asyncronously.  The lower consistency level (e.g. ONE), the faster you can write.  With lower consistency, you get better HA, also.
* Consistency level is defined at the app level.  So one app can have different consistency levels than other apps.
* You can define different replication factors per data center.  Data centers can be physically or logically divided.  Can have clients write to local data center and then data is async replicated to secondary data center.
* You can also have OLTP data center and another OLAP (reporting) data center.

### Writing to Cassandra

Writes are written to any node in the cluster (that node acts as a coordinator).

1. Writes to a commit log on disk.  Every write includes a timestamp.
1. Writes to a memtable (in memory table)
1. Memtables periodically flushed to disk called an sstable.
1. New memtable created in memory.
1. No deletes or updates in place occur.  Deletes are new tombstone records with a timestamp.  Memtable and sstables don't change over time.
1. A process called **compaction** periodically rolls many small sstables into larger sstables.  At this point, for any updated rows, only the latest values are stored in the new sstables.  Old versions of the row or any deleted rows are ignored and not written to the new sstables.
  
### Reading from Cassandra

Any node can be read from (acts as a coordinator).

1. Contacts nodes with the requested key.
1. On each node (node count based on replication factor), data is pulled from sstables and merged.  Any unwritten data in memtable is also merged in.
1. If nodes don't have same copy of the data, then the lastest data can sync to outdated nodes.  This is called read_repair_chance. 

Important that disk speeds will impact reads and compaction.  Compaction reduces the IO by reducing total number of files.

### Download/Install Cassandra

planetcassandra.org/cassandra - DataStax Distribution

### CQL

* CQL - Cassandra Query Language
* Keyspaces - container (similar to schema or database).  Primary purpose is to set replication factor.
* Tables - text, int, timestamp data types
* UUID - universal unique id
* TIMEUUID - first 64-bits is timestamp, and the rest is unique.
* Normal SELECT/FROM/WHERE queries.  No JOIN option.
* COPY table1 (col1, col2, col3) FROM 'tabledata.csv' WITH HEADER=true;

### Partitions

* First field in primary key is your partition key.
* In general, different partitions are located on different nodes.

### Clustering columns

* Any fields after the partition key.  So a primary key of (state, city) would make "state" your partition key and "city" your clustering column.  Clustering columns create sorting and uniqueness.
* "WITH CLUSTERING ORDER BY" clause on create table can also force sort order on disk for efficiency.
* "ALLOW FILTERING" allows you to specify a query without a partition key which scans the entire cluster.  **Don't use this.**

### Drivers

* JAVA, Python, C++, Ruby, etc.
* Connect to cluster (not a node)

### Node

* Can live in the cloud or on premise.
* Runs with a variety of disk types.
* SAN is **NOT** a good option.  Should be local storage.
* Each node can typically handle 3k-5k transactions per second per core.
* 1-3 TB of SSD/HDD per node should be the limit.
* Nodetool runs to show info/status to show info on the node and cluster (e.g. nodetool status).

### Adding a node to the cluster

* Communicates to the cluster and asks for data
* Schema/data and cluster topology synced to new node
* Node in "joining" status until synced and then it changes to "up" status
* Drivers know about which nodes have which data
* Can remove nodes in the same way.  They go into "leaving" status.

### Vnodes

Virtual nodes.  Allows for the partition keys to be virtually split up which allows for more even distribution of data and equal distribution of data onto a new node.

### Gossip

Cluster metadata (status of nodes - up/down or under stress) is shared between nodes asynchronously.  

* Application state is schared:
  * status (normal/joining/leaving)
  * data center location
  * rack location
  * schema version
  * disk load (at capacity??)
  * severity (IO pressure)
* Uses heartbeat timestamp to check who has the latest node information.  Updates cache with latest data of other nodes.

### Snitches

Added node has a config file (cassandra-rackdc.properties) with data center adn rack location.  That info is gossiped to other nodes.

### Replication

* You can replicate to multple nodes with replication factors.  RF=3 is best for production.
* You can also replicate to multiple data centers.  In your keyspace, you can specify "WITH REPLICATION = {'dc-west':3, 'dc-east':3}" where each data center gets three copies of the data.
* Data is written asynchronously.

### Consistency

* The **client** specifies the consistency level.
* Consistency Level (CL)
  * CL=ONE - only one node is required to be written/read from.
  * CL=QUORUM - more than 51% of nodes confirm write/read.
  * CL=ALL - all nodes confirm write/read.
* If you write and read with CL=one, you may write successfully to one node, but read from a different node before that other node has the new data.  
* If you use CL=QUORUM, then you solve the issue.  With RF=3, you'd confirm it was written to two nodes.  On any read after that, it would guarentee at least one node had the new data.
* Other CL options:
  * CL=LOCAL_QUORUM - allows you to get a quorum of just the local data center.  It will still sync data to other data centers as defined by your keyspace.
  * CL=EACH_QUORUM (writes only) - waits for a quorum in each data center.
  
### Hinted Handoffs

* Coordinator node will hold onto data until failed replica data nodes are back online.  
* Hint is saved for 3 hours by default.  If your node is down longer, you'll need to do a read repair.

### Read Repair

* Three types of Read Repair
  * On a CL=ALL or CL=QUORUM read by a client, if the data doesn't match, the latest data is returned to the client and then that data is synced to the out of date nodes.
  * On any read (even CL=ONE), read repair chance will (by default, 10% of the time) compare all nodes of that data and fix any inconsistencies by syncing the latest data.  Done in the background.
  * If you know data is out of sync (node down longer than 3 hours and the hinted handoff isn't kept), you can manually run "nodetool repair".  This syncs all data in the cluster.  
    * Run this on nodes that come back online.  
    * Run on nodes that aren't read from often.
    * **It's a good idea to run this on all nodes every 7-10 days to make sure the data is in sync.**

### Write Path

* Written to commit log on disk first.
* Written to memtable in RAM second.
  * Memtable is sorted based on primary key.
* Thirdly, user client is sent acknowledgement of write.
* After that, later, the memtables are written to disk as sstables.
  * The commit log segment is purged after written to disk.
  * The sstable is immutable (won't change).  
* **Ideally the commit log and sstable space would be on two separate disks.**
* Last write wins if two records of the same key exist.

### Read Path

* Needs to read from memtable and any sstables on disk.
* Sstables have a partition index to allow faster reads from a certain partition of data.
* On top of partition indexes there are summary indexes that are in memory mini indexes of the partition indexes in case the partition indexes grow too large.
* Bloom filters give a yes/no answer if a partition key is in the single sstable.  This allows the reader to skip lookups in the majority of the sstable files that don't contain the partition key.
* Key cache can jump you straight to a location of partition keys that are hit often.

Bloom Filter -> Key Cache -> Summary Index -> Partition Index -> sstable

### Compaction

* A background process that merges two sstables together.  Newer versions of the data are saved and older versions are thrown away.  Any deleted records (tombstones) are purged if beyond the grace period.  
* This process reduces the size and number of sstables and improves read performance.

### Adding Nodes (bootstrapping)

* Use virtual nodes.  This will allow you to add one node at a time.  
* **Wait 2 minutes between adding nodes.**  This allows the token ranges to be re-allocated and data to move around.
* 4 peices of information needs to be specified when adding a node specified in cassandra.yaml:
  * cluster_name - must match existing cluster name
  * rpc_address/listen_address - address of new node
  * -seeds - list of all existing nodes in the cluster
* When new node added:
  * Token range identified for new node.  Other nodes send data to new node.  New node in "joining" state until synced.  The more data, the longer it takes.  1 TB of data takes 15-20 minutes on a good network.
* Failure scenarios:
  * New node couldn't connect - check error log.  Fix config and try again.
  * Streaming fails - Run "nodetool rebuild" to resync.
* Optionially, you can run "nodetool cleanup" on the **other nodes** to read the sstables and remove data that was moved.  This can cleanup disk space but can cause load on the system.  Note that compaction will cleanup the data eventually, if you don't run this.

### Removing a node

* Removing an online node - run "nodetool decommission" command.  It will mark the node as "leaving" and start migrating data to other nodes.  After its done, the data will still reside on disk.  If you need to reclaim it, you'll need to delete teh data directories manually.
* Removing an offline node - run "nodetool removenode". This will adjust the tokens.  If this failes, run "nodetool assassinate".  After either works, run "nodetool repair" to make sure the data is consistent across the cluster.

### Replacing a down node

* Run "nodetool status" to find the IP address of the down node.  On the new node, edit the cassandra_env.sh script and swap the IP address of the dead node as the "replace_address" value in the JVM option.  This will enable bootstrapping of the new node.  
* Use "nodetool removenode" or "nodetool assassinate" to remove the dead node.  You can monitor the status with "nodetool netstats".  

### Running Repair Option

* Running "nodetool repair" will check the consistency of the data cross the cluster.  If the data isn't replicated as it should be, it will fix that.  
* **Incremental repairs** only look at sstable data since the last time the repair was ran.
* Running the repair **once a week** is a good idea.
* Running a repair on a subset of nodes at a time is good practice.
* Running repair if a node is down for more than 3 hours is required.
* If a node is down for a very long time, it is probably better to rebuild the node than repair it (too much data to sync).


