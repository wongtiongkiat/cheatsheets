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




