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

* Application state is shared:
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

### Dividing SSTables with sstablesplit

* If you did a nodetool compact and you have a really large SSTable, you can use sstablesplit to split large SSTable files into multiple smaller files.
* You **MUST** shut down Cassandra (the whole cluster) before running this.

### Creating Snapshots

* Snapshots will backup the data in case you have a castrophic failure or a user messed up the data and you need to recover.
* Creates a hard link of SSTables
  * Stored in a "snapshots" directory under each table directory.
* Incremental backups are disabled by default and can be turned on by setting incremental_backups to true in cassandra.yaml
  * An existing snapshot is needed before taking incremental backups
  * Incremental backups are stored in a "backups" directory under the keyspace data directory.
* Snapshots and incremental backups are stored locally.  Good for simple restores.  Bad for node/hardware failure.
* It's good practice to move these backups to an off-node location regularlly.  
  * Can use tablesnap to backup these files to S3
  * Can use cron with bash or rsync, etc to backup to other machines 
* Auto snapshot is on by default (shouldn't be disabled)
  * Will automatically snapshot before a table is truncated or tables/keyspaces are dropped.
* nodetool snapshot (needs to run on each node around the same time)
* nodetool clearsnapshot - will delete all snapshots
* Restore method:
  * Standard Method
    * Delete teh current data files and copy the snapshot and incremental files to the appropriate data directories
    * If using incremental backups, copy the contents of the backups directory to each table directory
    * Table schema must already be present in order to use this method
    * Restart and repair the node after the file copying is done
  * sstableloader Method
    * If you are loading into a different size cluster (backups from 10 nodes, loading into 6 nodes)
    * Must be careful, as it can add significant load to the cluster while loading
* Cluster wide backups
  * OpsCenter
  * Node backups synced on all nodes via SSH script
  
### Implementing Multiple Data Centers

* **Node** - virtual or physical host of single Cassandra instance
* **Rack** - logical grouping of physically related nodes
* **Data Center** - a logical or physical grouping of a set of racks
* Enables geographically aware read/write request routing
* Each node belongs to one rack in one data center
* The node's rack/data center is identified in conf/cassandra-rackdc.properties

* Why have multiple data centers?
  * Continuous availability of the data/app
  * Live backup
  * Improved performance (reduces latency for users)
  * Analytics
  
* What if one data center goes down?
  * Failure of a data center will likely go unnoticed by end users
  * If node/nodes fail, they will stop communicating via gossip
  * Recovery can be accomplished with a rolling repair to all nodes in failed data center

* How to implement multi-data center cluster
  * Use the NetworkTopologyStrategy rather than SimpleStrategy
  * Use LOCAL consistency level for read/write operations to limit latency
  * If possible, define one rack for entire cluster
  * Specify the snitch

### Best Practices for Cluster Sizing

* Estimating the volume of data
  * Data per node (avg amount of data per row) x (number of rows) / (number of nodes)
* Velocity of data
  * How many writes per second?
  * How many reads per second?
  * Volume of data per operation?
* Factor in replication
  * Multiply per node volume by Replication Factor (RF=3 means three copies of the data)
* Testing limits
  * Use cassandra-stress to simulate workload
  * Test production level servers
  * Monitor to fin dlimits
    * Disk limits
    * CPU limits
* Example
  * Assuming:
    * 100 GB of data per day
    * Writes - 150k per second
    * Reads - 100k per second
    * Replication Factor = 3
    * 2 data centers
    * 25ms read latency 95th percentile
  * Testing summary
    * Node capability to maintain < 25ms reads
      * At maximum packet size, 50k writes/sec
      * At maximum packet size, 40k reads/sec
      * 4 TB per node to allow for compaction overhead
    * Sizing for Volume
      * (100 GB per day) x (30 days) x (6 months retention) = 18 TB data
      * (18 TB data) x (RF 3) = 54 TB total cluster load
      * (54 TB total data) / (4 TB max per node) = 14 nodes
      * (14 nodes) x (2 data centers) = 28 total nodes
    * Sizing for Velocity
      * (150k writes/sec load) / (50k writes/sec per node requirement) = 3 nodes
      * Since 3 nodes is less than 14 nodes, our volume capacity outranks the velocity capacity.  So 14 nodes per data center.
    * Future Capacity
      * Validate assumptions often
      * Monitor for changes over time
      * Plan for increasing cluster size before you need it
      * Be ready to reduce cluster size if needed (you over-estimated)
      
### Using the cassandra-stress Tool

* Java based stress testing utility for Cassandra
* Uses a YAML profile to define a specific schema with compaction strategy
  * DDL - defining your schema
  * Column Distribution - for defining shape and size of each column globally and within each partition
  * Insert Distributions - for defining how the data is written during the stress test
  * DML - for defining how the data is queried during the stress test
  
### Using CQL Copy command

* Import/Export delimited data to/from Cassandra
* Some rules:
  * Every row in the delimited input file contains the same number of columns in the table.
  * Empty data for a column is assumed by default as a NULL value
  * COPY FROM is intended for importing small datasets (a few million rows or less).  It isn't meant as a backup strategy.
  * For importing larger datasets, use Cassandra bulk loader.
* COPY FROM Example:
  * COPY killrvideo.users FROM 'users.txt' WITH DELIMITER='|' AND HEADER=TRUE
* Options:
  * DELIMITER - set the character (single character) that the data is delimited with (default is comma)
  * HEADER - set true to inidcate the first row in the file is the header (default is false)
  * CHUNKSIZE - set the size of chunks passed to worker processes (default is 1000)
  * SKIPROWS - the number of rows to skip (default is 0)
* COPY TO Example:
  * COPY users (firstname, lastname, created_date, email) TO 'users.csv'
* You may also use STDIN or STDOUT keywords to import from standard input or export to standard output.  

### Stream Data with sstableloader

* Provides ability to bulk load external data into Cassandra
* You can also use it to load pre-existing SSTables into an existing or new cluster
* Streams table data back into the cluster
* Example:
  * sstableloader -d 110.82.155.1 /var/lib/cassandra/data/killrvideo/users/

### Explore your SSTables with sstabledump

* Allows you to see the raw data of a SSTable in a text format (dumps to JSON format)
  * You can see things like timestamps and tombstones as well

### Import data with Spark

* Provides convienient functionality for loading large external data sets into Cassandra
* Ingests files in CSV, TSV, JSON, XML, and other formats
* Can do some validation and transformations on the data before loading into Cassandra

### Configuring the cassandra.yaml file

* This is the main configuration file for Cassandra
  * Default location is /etc/cassandra
* Minimum properties
  * cluster_name - must be the same name as each of the other nodes in the cluster (default: Test Cluster)
  * listen_address - network ip address on the node (default: localhost)
  * listen_interface - network port (default: eth0)
  * listen_interface_prefer_ipv6 - IP v6 flag (default: false)
* Storage options
  * hinted_handoff_enabled - hinted handoff is performed when set to true
  * max_hint_window_in_ms - this defines the maximum amount of time a dead host will have hints generated
  * data_file_directories - the location of data files
  * commitlog_directory - the location of the commit log directory
  * row_cache_size_in_mb - maximum size of the row cache in memory
* Communications
  * partitioner - responsible for distributing groups of rows (by partition key) across nodes in the cluster
  * storage_port - TCP port for commands and data
  * broadcast_address - address to broadcast to other Cassandra nodes
* Internal node configuration
  * concurrent_reads/concurrent_writes - number of reads/writes permittedto occur concurrently
  * file_cache_size_in_mb - maximum memory to use for pooling sstable buffers
  * memtable_heap_space_in_mb/memtable_offheap_space_in_mb - total on heap and off allowance for membtables
* Security
  * authorizer - the authentication backend.  It implements IAuthenticator for identifying users
  * internode_authenticator - used to allow/disallow connections from peer nodes
  
### System & Output logs

* Simple Logging Facade for Java
* A logback backend
* Logs written to system.log and debug.login (in logs directory)
* Configure logging programmatically or manually
  * manually
    * Run nodetool setlogginglevel command
    * Configure the logback-test.xml or logback.xml file installed with Cassandra
    * Use the JConsole tool to configure logging through JMX
 
### Using the nodetool utility

* Command line interface for interacting with your Cassandra node
* It's a command wrapper for JMX interactions
* Groups of commands:
  * Cluster
    * nodetool status - provides information about the cluster, such as state, load, and IDs
  * Server
    * nodetool repair - starts a repair process from the point of view of that node. repairs one or more tables
    * nodetool info - specific information about the node you are on. Load, uptime, status of JVM
    * nodetool tpstats - provides usage stats on thread pools.  How many completed, pending, blocked?  A high number of pending tasks can indicate performance problems.  Shows mutation drops (critical for troubleshooting a node)
    * nodetool tablehistograms - provides statistics on a table.  Includes read/write latency, partition size, column count, and number of SSTables.  The report is incremental, not cumulative.  It covers all operations since the last time nodetool tablehistograms was run in the current session
  * Backup
    * nodetool snapshot - takes a snapshot of one or more keyspaces, or a table, to backup data.  To backup an entire cluster, you would need to take a snapshot on all nodes around the same time
    * nodetool clearsnapshot - removes one or more snapshots
  * Storage
    * nodetool cleanup - used to get rid of old data on nodes after bootstrap operations.  Deletes snapshots in one or more keyspaces.  Not specifying a snapshot name removes all snapshots
    * nodetool flush - flushes everything in memtables out to SSTables and deletes all commit log segments.  Can use before shutting down a node to write all data to SSTables.  This can speed up restart of the app
  * Compaction
    * nodetool compact - forces a major compaction on one or more tables.  **Don't do this unless you know what you are doing**
    * nodetool compactionstats - provies stats about compaction. See what is currently compacting
  * Network
    * nodetool proxyhistograms - provides a histogram of network stats.  Includes percentile rank of read/write latency values
    * nodetool netstats - provides network information about a host
  
### Monitoring Your Cluster with OpsCenter

* OpsCenter is built by DataStax.  Performance, Backup/Restore, Repair, and Best Practices services
* Visual Monitoring and Management
  * Control automatic management services including transparent repair
  * Managed and schedule backup and restore operations
  * Perform capacity planning with historical trend analysis and foecasting capabilities
  * Proactively manage all clusters with threshold and timing-based alerts
  * Visually create new clusters with a few mouse clicks
  * Built-in Automatic Failover
* Performance Service
  * Collects key performance metrics to quickly troubleshoot cluster performance
  * Analyzes Query, Table, and Cluster specific metrics
  * Correlates diferent metrics and provides custom recommendations
  * Eliminates the need for custom scripting and scheduling to detect problem nodes
* Backup/Restore Service
  * User-configuarable
  * Optimized storage
  * Automatic Cleanup
  * Store backups in AWS S3 (offsite)
  * Coordinated Restore
  * Supports different Topology
  * Cloning
  * Point In Time Restore
* Visual Repair Service
  * Automatically maintains data consistency across a cluster without impacting performance
  * Ensures that your cluster operates efficiently by optimally running repairs
* Best Practice Service
  * Periodically scans database clusters and automatically detects issues that threaten a cluster's security, availability, and performance
  * Utilizes a set of expert rules that span a variety of different categories and summarizes the results
  
### Intro to JMX

* Java Management Extensions
* Gathering metrics from Cassandra is done through JMX
* JMX has MBeans (Management Beans).  Each metric is a MBean.  Found under org.apache.cassandra.metrics
  * domain:key=value,key2=value2

### Leveled Compaction

* A process to create multiple levels of SSTables through compaction
* Leveled compaction is good for high read and low write systems due to the overhead on IO for the frequent compaction
* Leveled compaction uses a multiplier of 10 per level.  So the default SSTable max size of 160MB would cause Level 1 limit to be 1600MB.  Level 2 limit would be 16,000MB, and so on
* Each partition only resides in one SSTable per level max
* Reads handled by only a few SSTables, which makes it faster
* 90% of the data resides in the lowest level (due to the 10x rule), unless the lowest level isn't full
* Pros:
  * Great for high read and lower write environments
  * Wastes less disk space
  * Grouping makes updated records merge more frequently with older records
* Cons: 
  * Frequent compactions causes higher IO
  * Compacts more SSTables at once over size tiered compaction
  * Can't injest data at high insert speeds

### Size Tiered Compaction

* A compaction process to group similiarly sized SSTables together
* Tiers with less than min_threshold (four) SSTables are not considered for compaction
* The smaller the SSTables, the "thinner" the distance between min_threshold and max_threshold
* SSTables qualifying for more than one tier are distributed randomly amongst buckets
* Buckets with more than max_threshold SSTables are trimmed to just that many SSTables (32 by default.  SSTables least hit will be dropped into new bucket)
* The tier that is accessed the most (the hottest) is chosen for compaction first
* Similar sized SSTables compact together better
* SSTables of similar size will have a fair amount of overlap
* concurrent_compactors - allows you to set parallel compactors
* When does Compaction run?
  * When MemTable flushes to SSTable
    * MemTable too large
    * Commit Log too large
    * Manual flush
  * Cluster streams SSTable segments to new node
    * Bootstrap
    * Rebuild
    * Repair
* Compaction continues until there are no more tiers/buckets with at least min_threshold tables in it
* Size Tiered Compaction is the default compaction for tables
* Pros:
  * Great for high write by procrastinating compaction as long as possible
  * compaction_throughput_mb_per_sec - controls the compaction IO load on a node
* Major compaction will compact all SSTables into one SSTable **Don't do this**  

### Time Windowed Compaction

* Built for Time Series Data
* Each time window is compacted into a single SSTable
* compaction_window_unit - minutes, hours, days
* compaction_window_size - Number of units in each window
* expired_sstable_check_frequency_seconds - determines how often to check for fully expired (tombstoned) SSTables

### Configuring JVM Settings

* JVM is the Java Virtual Machine that Cassandra runs within.  Each JVM has CPU and Memory/Heap resources
* cassandra-env.sh - controls environment settings, such as JVM configuration settings for Cassandra as well as JMX options
  * MAX_HEAP_SIZE - If less tan 2GB of system memory, it uses 1/2.  If 2-4GB of system memory, it will use 1GB.  If more than 4GB of system memory, it will use 1/4, but won't exceed 8GB.  Large heaps can introduce Garbage Collection pauses that lead to latency
  * HEAP_NEWSIZE - 100MB per core.  The larger this is, the longer Garbage Collection (GC) pause times will be.  The shorter it is, the more expensive GC will be
* jvm.options - garbage collection settings **Don't change unless you know what you're doing**
* Tuning Java Garbage Collection
  * Default is Concurrent-Mark-Sweep (CMS) garbage collector
  * G1 garbage collector is better than CMS for large heaps
    * Goes for regions of heap with most garbage first
    * Compacts the heap on-the-go
  * Garbage collection is configured in conf/jvm.options
    * Comment out CMS configuration lines and Uncomment the G1 settings, and restart process
* You can also view JVM settings through JMX on port 7199

### Understanding Garbage Collection

* It cleans up unused memory to free memory
* Minimize the length of pauses and increase throughput on the amount of memory it can clean up per cycle
* A full garbage collection (full GC) is bad because it pauses the application while it cleans up memory.  If garbage collection can't clean out memory fast enough, it will trigger a full GC

### Generating Heap Dumps for Troubleshooting

* Useful when troubleshooting high memory utilization or OutOfMemoryErrors
* Shows exactly which objects are consuming most of the heap
* Out of Memory errors are automatically dumped
* By default, Cassandra puts the dump file in a subdirectory of the working, root directory when running as a service
* You can set a new dump path by changing cassandra-env.sh and setting CASSANDRA_HEAPDUMP_DIR to the new path
* Can manually trigger a heap dump using "sudo -u user jmap -dump:file=filename,format=b pid"
* Eclipse Memory Analyzer Tool (MAT) is a tool that can be used to analyze a heap dump

### JVM Profiling

* How to discover issues with the JVM
  * nodetool tpstats
    * Can see Active, Pending, Blocked, All time blocked threads
    * Blocked threads are problematic
    * Mutations that are dropped are also bad
  * OpsCenter
    * Graphing JVM stats
  * JMX Clients
    * jconsole
    * visualvm

### Importance of Time Sync

* Time Stamp Counter (TSC) is the most common clock source. 
* The clocks on all nodes should be synchronized.  If they are out of sync, it can lead to unstableness
* You can use NTP (Network Time Protocol) or other methods

### Configuring the Kernel

* limits.conf
``` text
  * - nofile      1048576
  * - memlock     unlimited
  * - fsize       unlimited
  * - data        unlimited
  * - rss         unlimited
  * - stack       unlimited
  * - cpu         unlimited
  * - nproc       unlimited
  * - as          unlimited
  * - locks       unlimited
  * - sigpending  unlimited
  * - msgqueue    unlimited
```
* Turn off swap
  * swapoff -a
  * remove all swap files from /etc/fstab
``` text 
    sed -i 's/^\(.*swap\)/#\1/' /etc/fstab
```
* sysctl.conf
``` text
   net.ipv4.ip_local_port_range = 10000 65535
   net.ipv4.tcp_window_scaling = 1
   net.ipv4.tcp_rmem = 4096 87380 16777216
   net.ipv4.tcp_wmem = 4096 65536 16777216
   net.core.rmem_max = 16777216
   net.core.wmem_max = 16777216
   net.core.netdev_max_backlog = 2500
   net.core.somaxconn = 65000
```

### Useful Linux Observation Tools
