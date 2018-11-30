## Chapter 2

#### Server architect

```
[root@MyCentOS ~]# ps f -U postgres
  PID TTY      STAT   TIME COMMAND
 1918 tty1     S      0:00 /usr/local/pgsql/bin/postgres
 1920 ?        Ss     0:00  \_ postgres: checkpointer process   
 1921 ?        Ss     0:00  \_ postgres: writer process     
 1922 ?        Ss     0:00  \_ postgres: wal writer process   
 1923 ?        Ss     0:00  \_ postgres: autovacuum launcher process 
 1924 ?        Ss     0:00  \_ postgres: stats collector process   
 ```
 
`/usr/local/pgsql/bin/postgres` starts utility processes such as `bgwriter, checkpointer, autovacuum launcher, log writer, stats collector process,`

`postgres` daemon flow:

![postgres_daemon_process](https://github.com/wongtiongkiat/cheatsheets/blob/master/Postgres%20for%20Data%20Architect/img/postgres_daemon_process.jpg)

![postgres_daemon_process_2.jpg](https://github.com/wongtiongkiat/cheatsheets/blob/master/Postgres%20for%20Data%20Architect/img/postgres_daemon_process_2.jpg)

#### Shared buffer
Amount of memory reserved for shared memory dedicated in `postgresql.conf`, used for caching. Note that there are many **other** caches, such as `OS cache`, `disk controller cache`, `disk drive cache` and etc. Large cache may not necessary help as bigger cache usually mean more time to flush into physical drive.

![database_cache_diagram.jpg](database_cache_diagram.jpg)

* Database will check for whether if the data exists in the buffer, thus reducing physical I/O load.
* Most data related fetches/writes will happen in the buffer.
* A commit will ensure that Write Ahead Log (WAL) will be synchronized with the WAL buffer.

Check on the book for knowing how to read shared buffer

#### Checkpoint
* Postgres always read/write data in blocks (also known as page) in 8K bytes 

* Check on the book for knowing how to check size of blocks/pages

* It is the job of checkpointer process to write dirty buffers (changes to the data that are not written to the data files) into the data file. When a checkpoint happen all dirty pages are written to tables and index files. WAL log is applied up to this checkpoint**.

![pgdata_directories_pages.jpg](pgdata_directories_pages.jpg)

* Image above shows that for each database, we have files for tables where within the files data are organized in blocks.


##### Illustrate checkpoint
1. Insert some data
```
INSERT INTO emp(id , first_name) SELECT generate_series(1,5000000), 'A longer  name ';
INSERT 0 5000000
```
2. After executing the preceding query a few times, let's check the size of the files from the shell prompt:
```
[root@MyCentOS 24741]# ls -lh 24742*
-rw-------. 1 postgres postgres 1.0G Nov 17 16:14 24742
-rw-------. 1 postgres postgres  42M Nov 17 16:14 24742.1
-rw-------. 1 postgres postgres 288K Nov 17 16:08 24742_fsm
-rw-------. 1 postgres postgres  16K Nov 17 16:06 24742_vm
```


#### WAL logs

* [From postgres documentation](https://www.postgresql.org/docs/10/sql-checkpoint.html): A checkpoint is a point in the write-ahead log sequence at which all data files have been updated to reflect the information in the log. All data files will be flushed to disk.
* At checkpoint time all data are flushed onto the disk, and a **special checkpoint** is written to the log file. 
* When crash recovery procedure happens, it looks at the latest **special checkpoint** from which `REDO` operation should happen.
* After a checkpoint, previous log segments before the checkpoint are no longer needed, thus can be discarded/removed as free space.

#### When determine checkpoint to occur?

* checkpoint_segments
* checkpoint_timeout
* checkpoint_completion_target

#### checkpoint_segments

* default -> 3 checkpoint_segments, each WAL segment is 16 MB, once 3 WAL segments of changes has been made, checkpoint will occur.

#### checkpoint_timeout

* time period has been elapsed, when timeout happens the checkpoint will occur.

#### checkpoint_completion_target

* how quickly should checkpointing process finish in each iteration => default value 0.5, when let's say the value increased to 0.9 it will spread the checkpoint over to a longer period.

#### WAL Buffer and the WAL writer process

* Changes are first made to WAL buffer, then flushed onto WAL segment each in 16MB sizes

```
[postgres@MyCentOS pg_xlog]$ pwd
/pgdata/9.3/pg_xlog
[postgres@MyCentOS pg_xlog]$ ls -alrt
total 16396
drwx------.  2 postgres postgres     4096 Oct 13 13:23 archive_status
drwx------.  3 postgres postgres     4096 Oct 13 13:23 .
drwx------. 15 postgres postgres     4096 Nov 15 20:17 ..
-rw-------.  1 postgres postgres 16777216 Nov 15 20:17 000000010000000000000001
```

* there are functions related to WAL used for archival, recovery and etc, example: `pg_switch_xlog` moves to the next transaction log file.
* WALs are mostly writes and rarely read when wal is read it usually is because of : `recovery`, `server startup`, `replication`

#### Recovery

* WAL's primary concept is to recover transactions have been commited but not yet updated in the database file system. All changes made to the database will be recorded in WAL segments.
* Loss of WAL files means loss of transactions.

#### Incremental backup and point-in-time recovery

* Snapshot of database filesystem (postgres filesystem) can be taken, and followed by setup of WAL archival process.
* The snapshot does not have to be consistent, as we can always just use the snapshot and perform point-in-time recovery (replay WAL segments until a specific transaction)


#### Replication

* Since all changes happened in the database will be recorded in WAL segment, why not just use those to get a stand-by server ready for failover.
* WAL can also be thought of as a cushion between the actual data and WAL buffer, so that a transaction does not result in immediate data write.

#### Key parameters related to WALs
```
[postgres@MyCentOS 9.3]$ grep wal postgresql.conf 
wal_level = archive             # minimal, archive, or hot_standby
#wal_sync_method = fsync        # the default is the first option
#wal_buffers = -1               # min 32kB, -1 sets based on shared_buffers
#wal_writer_delay = 200ms       # 1-10000 milliseconds
#max_wal_senders = 0            # max number of walsender processes
#wal_keep_segments = 0          # in logfile segments, 16MB each; 0 disables
#wal_sender_timeout = 60s	# in milliseconds; 0 disables
#wal_receiver_status_interval = 10s   # send replies at least this often
#wal_receiver_timeout = 60s           # time that receiver waits for
```

