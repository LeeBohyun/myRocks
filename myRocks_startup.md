# MySQL/MyRocks Installation and TPC-C/Sysbench testing: 

**:warning: experiment setup based on Ubuntu 16.04.**

## How to install MyRocks

### Prerequisites

```bash
$ sudo apt-get update
$ sudo apt-get install g++ cmake libbz2-dev libaio-dev bison \
  zlib1g-dev libsnappy-dev libgflags-dev libreadline6-dev libncurses5-dev \
  libssl-dev liblz4-dev libboost-dev gdb git
```

- libcap-dev
```bash
$ sudo apt-get install libcap-dev
```

- zstd library
Download zstd library from [this link](https://github.com/facebook/zstd):
```bash
$ cd zstd-1.4.5
$ make install
```
you can check zstd library with below command line.
```bash
$ zstd --version
```
### Setup Git Repository

```bash
$ git clone https://github.com/facebook/mysql-5.6.git
$ cd mysql-5.6
$ git submodule init
$ git submodule update

$vim ~/.bashrc
...
export PATH=$PATH:/home/lbh/mysql-5.6/bin:/home/lbh/mysql-5.6/scripts
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/lbh/mysql-5.6/lib/
... :wq
$ source ~/.bashrc

$ cmake . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_SSL=system -DWITH_ZLIB=bundled -DMY_SQL_MAINTAINER_MODE=0 -DENABLED_LOCAL_INFILE=1 -DENABLE_DTRACE=0 -DCMAKE_CXX_FLAGS="-march=native" -DDOWNLOAD_BOOST=ON -DWITH_BOOST=/home/lbh/mysql-5.7.24 -DCMAKE_INSTALL_PREFIX=/home/lbh/mysql-5.6
$ sudo make -j8 install
```
### Example myrocks.cnf
```bash
$ vim my.cnf
[mysqld]
rocksdb
default-storage-engine=rocksdb
skip-innodb
default-tmp-storage-engine=MyISAM
collation-server=latin1_bin
log-bin
binlog-format=ROW

socket=/tmp/mysql.sock
port=3306
datadir=/home/lbh/test_data

rocksdb_max_open_files=-1
rocksdb_max_background_jobs=8
rocksdb_max_total_wal_size=4G
rocksdb_block_size=16384
rocksdb_table_cache_numshardbits=6
rocksdb_block_cache_size=10G

# rate limiter
rocksdb_bytes_per_sync=4194304
rocksdb_wal_bytes_per_sync=4194304
rocksdb_rate_limiter_bytes_per_sec=104857600

# triggering compaction if there are many sequential deletes
rocksdb_compaction_sequential_deletes_count_sd=1
rocksdb_compaction_sequential_deletes=199999
rocksdb_compaction_sequential_deletes_window=200000

rocksdb_default_cf_options=write_buffer_size=128m;target_file_size_base=32m;max_bytes_for_level_base=512m;level0_file_num_compaction_trigger=4;level0_slowdown_writes_trigger=10;level0_stop_writes_trigger=15;max_write_buffer_number=4;compression_per_level=kLZ4Compression;bottommost_compression=kZSTD;compression_opts=-14:1:0;block_based_table_factory={cache_index_and_filter_blocks=1;filter_policy=bloomfilter:10:false;whole_key_filtering=1};level_compaction_dynamic_level_bytes=true;optimize_filters_for_hits=true;compaction_pri=kMinOverlappingRatio

rocksdb_wal_dir=/home/lbh/test_log

rocksdb_use_direct_io_for_flush_and_compaction=ON
rocksdb_use_direct_reads=ON
```

## MySQL initialize
```bash
$ sudo chmod +x scripts/mysql_install_db
$ ./scripts/mysql_install_db --defaults-file=/home/lbh/myrocks.cnf
```

Start the server:

```bash
$ ./bin/mysqld_safe --defaults-file=/home/lbh/myrocks.cnf
```

Shutdown the server:

```bash
$ ./bin/mysqldadmin -uroot shutdown
```
##  MySQL Root Password 

Set MySQL root password:

```bash
$ ./bin/mysqld_safe --defaults-file=/home/lbh/myrocks.cnf --skip-grant-tables --datadir=/home/lbh/test_data
$ ./bin/mysql -uroot -S/tmp/mysql.sock -P3306

root:(none)> use mysql;
root:mysql> update user set authentication_string=password('evia6587') where user='root';
root:mysql> flush privileges;
root:mysql> quit;

$ ./bin/mysql -uroot 

root:mysql> set password = password('yourPassword');
root:mysql> quit;
```

## TPC-C Testing
Since MyRocks ahve different storage engine and does not support foreign key, we need to create new sql file and load data. 

```bash
$ cat create_table.sql | sed -e "s/Engine=InnoDB/Engine=RocksDB DEFAULT COLLATE=latin1_bin/g" > create_table_myrocks.sql
$ grep -v "FOREIGN KEY" add_fkey_idx.sql > add_fkey_idx_myrocks.sql 
```
Change ```load.sh``` 
```bash

```
Do rest of the steps [likewise](https://github.com/LeeBohyun/mysql-tpcc/blob/master/multi-mysql-tpcc.md).

## Sysbench Testing

-Data Load:
```bash
$ sysbench --test=/home/lbh/sysbench-1.0.12/tests/include/oltp_legacy/oltp.lua --mysql-host=localhost  --mysql-db=sbtest --mysql-user=root --mysql-password=evia6587 --max-requests=0  --oltp-table-size=2000000 --max-time=600  --oltp-tables-count=200 --report-interval=10 --db-ps-mode=disable  --random-points=10 --mysql-table-engine=rocksdb --mysql-socket=/tmp/mysql.sock1 --mysql-port=3307 --num-threads=128  prepare
```

-Sysbench Testing:
```bash
$ sysbench --test=/home/lbh/sysbench-1.0.12/tests/include/oltp_legacy/oltp.lua --mysql-host=localhost  --mysql-db=sbtest --mysql-user=root --mysql-password=evia6587 --max-requests=0  --oltp-table-size=2000000 --time=259200 --oltp-tables-count=200 --report-interval=10 --db-ps-mode=disable  --random-points=10 --mysql-table-engine=rocksdb --mysql-socket=/tmp/mysql.sock1 --mysql-port=3307 --num-threads=128  run |tee /home/lbh/result/sb-ns.txt
```

