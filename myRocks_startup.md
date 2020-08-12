# MySQL/MyRocks Installation and TPC-C testing: 

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

$ cmake . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=0 -DENABLED_LOCAL_INFILE=1 -DCMAKE_INSTALL_PREFIX=/home/lbh/mysql-5.6
$ time make
$ sudo make install && make clean
```
## TPC-C Testing
Since MyRocks ahve different storage engine and does not support foreign key, we need to create new sql file and load data. 

```bash
$ cat create_table.sql | sed -e "s/Engine=InnoDB/Engine=RocksDB DEFAULT COLLATE=latin1_bin/g" > create_table_myrocks.sql
$ grep -v "FOREIGN KEY" add_fkey_idx.sql > add_fkey_idx_myrocks.sql 
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

