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
export PATH=/home/lbh/mysql-5.6/bin:$PATH                                                     │·····································
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/lbh/mysql-5.6/lib/ 
... :wq
$ source ~/.bashrc

$ cmake . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=0 -DENABLED_LOCAL_INFILE=1 -DCMAKE_INSTALL_PREFIX=/home/lbh/mysql-5.6
$ time make
$ sudo make install && make clean
```
