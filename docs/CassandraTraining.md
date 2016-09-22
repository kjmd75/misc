## Cassandra Training

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

