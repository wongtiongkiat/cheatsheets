## Chapter 1

Postgres installation files will have 4 directories:
1. `include` => contains header files for extensions
2. `lib` => contains libraries to be dynamically linked
3. `share` => default configuration files when cluster is initialize, extensions, documentation
4. `bin` => contains binary files (createdb, clusterdb, initdb, psql, postgres, ...)

#### Cluster initialization
1. `adduser postgres` => does not have to be postgres, can be any user
2. `mkdir -p /paths`
3. `chown postgres` /paths
4. `su -u postgres` => substitute user to postgres
5. `initdb --pgdata=/pgdata/9.3 --pwprompt` => pgdata path to store the data, pwprompt flag asks for password

#### Cluster walkthrough
* `pg_ctl` is a utility to start, stop, check the status, or restart the database cluster
* `base directory` holds databases created by user, notice that `Oid points` to the directory names
```
[postgres@MyCentOS base]$ pwd
/pgdata/9.3/base
[postgres@MyCentOS base]$ find ./ -type d
./
./1
./12891
./12896
[postgres@MyCentOS base]$ oid2name
All databases:
Oid      |   Database Name |  Tablespace
---------|-----------------|--------
  12896  |   postgres      |  pg_default
  12891  |   template0     |  pg_default
      1  |    template1    |  pg_default
```
* `global directory` keep track of entire cluster, database roles, system catalog data, book-keeping and etc.
* `pg_clog directory` contains transaction commit status data
* `pg_multixact directory` contains concurrent transactions lock statuses
* `pg_serial directory` serializable transactions
* `pg_snapshots directory` exported snapshots
* `pg_stat_tmp directory` statistics table
* `pg_subtrans directory` subtransaction statuses
* `pg_tblspc directory` symbolic links to tablespace
* `pg_twophase directory` state files
* `pg_xlog` directory containing WAL files

#### Processes
`ps  -fupostgres` => show parent process
```
UID      | PID  |  PPID | C | STIME | TTY | TIME | CMD
postgres | 1566 | 1     | - | ----- | --- | ---- | /usr/local/pgsql/bin/postmaster -D /pgdata/9.3
postgres | 1666 | 1566  | - | ----- | --- | ---- | ---
postgres | 1667 | 1566  | - | ----- | --- | ---- | ---
postgres | 1668 | 1566  | - | ----- | --- | ---- | ---
postgres | 1669 | 1566  | - | ----- | --- | ---- | ---
postgres | 1670 | 1566  | - | ----- | --- | ---- | ---
postgres | 1671 | 1566  | - | ----- | --- | ---- | ---
```
```
[postgres@MyCentOS 9.3]$ pg_ctl status
pg_ctl: server is running (PID: 1566)
[postgres@MyCentOS 9.3]$ head -1 postmaster.pid
1566
```
[postmaster is an obsolete alias for postgres](https://dba.stackexchange.com/questions/102453/why-have-both-a-postmaster-and-postgres-executable)
`/usr/local/pgsql/bin/postmaster` does the following:
* the main process for starting out other processes
* one postmaster always manages the data from exactly one database cluster
* each cluster will have its own `postmaster.pid`
* the very first line of `postmaster.pid` should be the parent process

#### Extensions

```
postgres=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description
---------+---------+------------+------------------------------
plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)
```

```
postgres=# SELECT name,comment  FROM pg_available_extensions limit 5;
   name   |                           comment
----------+--------------------------------------------------------------
dblink   | connect to other PostgreSQL databases from within a database
isn      | data types for international product numbering standards
file_fdw | foreign-data wrapper for flat file access
tsearch2 | compatibility package for pre-8.3 text search functions
unaccent | text search dictionary that removes accents
(5 rows)
```

##### Install extension

```
postgres=# CREATE EXTENSION dblink ;
CREATE EXTENSION
postgres=# \dx
                         List of installed extensions
  Name   | Version |   Schema   |                         Description
---------+---------+------------+--------------------------------------------------------------
dblink  | 1.1     | public     | connect to other PostgreSQL databases from within a database
plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)
```

##### Drop extension

```
postgres=# DROP EXTENSION dblink ;
DROP EXTENSION
postgres=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description
---------+---------+------------+------------------------------
plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)
```
