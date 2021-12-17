---
layout: post
title: "压测基于 Patroni 的 Citus 高可用集群"
date: "2021-05-09 22:10:00"
category: PostgreSQL
tags: db PostgreSQL
author: lework
---
* content
{:toc}

## 集群环境

见 [基于 Patroni 的 Citus 高可用环境部署](/2021/05/08/patroni-citus-ha/)

## 参数修改

默认参数




## 压测软件

使用 pgbench

`pgbench`程序会创建4张表：

 - `pgbench_branches`
 - `pgbench_tellers`
 - `pgbench_accounts`
 - `pgbench_history`

选项参数介绍：
`-i, --initialize`, 进入初始化模式
`-F, --fillfactor=NUM`， 设置填充因子，默认值为100
`-n, --no-vacuum`， 初始化结束后不执行VACUUM操作，（注：VACUUM操作其实就是整理磁盘碎片空间）
`-q, --quiet`，开启静默模式，打印少量信息
`-s, --scale=NUM`， 生成数据的比例因子，默认值1，值越大，表里初始化的数据就越多。当值为k时，各个表的数据量如下所示：

- `pgbench_branches`   1K
- `pgbench_tellers`    10K  
- `pgbench_accounts`   100000*K
-  `pgbench_history`    0

`-T`,测试执行的时间，单位是秒
`-t`,设定每个客户端运行多少数量的事务后结束
`-c`,模拟客户端的数量，也就是连接数量
`-C`，为每个事务创建new connection
`--foreign-keys`， 在初始化的表之间创建外键
`--index-tablespace=TABLESPACE`，指定索引表空间
`--tablespace=TABLESPACE`， 指定表空间

## 压测环境1

| hostname |           ip | os         | 配置 |
| -------- | -----------: | ---------- | ---- |
| test     | 10.10.10.138 | CentOS 7.8 | 2C4G |

**主要软件**

- CentOS 7.8
- PostgreSQL `12.6`

### 本地表测试

**生成三千万条数据**

```
/usr/pgsql-12/bin/pgbench -i -s300 -h 10.10.10.140 -p 5432 -U postgres -d postgres
```


**只读测试**

```
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres

...
transaction type: <builtin: select only>
scaling factor: 300
query mode: simple
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 384083
latency average = 24.984 ms
latency stddev = 37.246 ms
tps = 3195.576593 (including connections establishing)
tps = 3197.448319 (excluding connections establishing)
statement latencies in milliseconds:
         0.012  \set aid random(1, 100000 * :scale)
        24.961  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        
```

**读写测试**

```bash
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres

...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 38130
latency average = 251.683 ms
latency stddev = 329.592 ms
tps = 316.864187 (including connections establishing)
tps = 317.109667 (excluding connections establishing)
statement latencies in milliseconds:
         0.089  \set aid random(1, 100000 * :scale)
         0.077  \set bid random(1, 1 * :scale)
         0.067  \set tid random(1, 10 * :scale)
         0.062  \set delta random(-5000, 5000)
         0.797  BEGIN;
        86.515  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.609  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         2.082  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        15.103  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.441  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       145.878  END;
```



### 分布式表(3个shard)测试

创建分布式表，表用上面的

```plsql
--将tpc-b涉及的4张表转换为sharding表（3个shard,每台物理机每个PG实例1个shard）
set citus.shard_count=3;
select create_distributed_table('pgbench_accounts','aid');
select create_distributed_table('pgbench_branches','bid');  
select create_distributed_table('pgbench_tellers','tid');  
select create_distributed_table('pgbench_history','aid');  

-- 删除本地数据

SELECT truncate_local_data_after_distributing_table($$public.pgbench_accounts$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_branches$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_tellers$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_history$$);
```



**只读测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
transaction type: <builtin: select only>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 929166
latency average = 10.225 ms
latency stddev = 35.107 ms
tps = 7734.042196 (including connections establishing)
tps = 7742.881072 (excluding connections establishing)
statement latencies in milliseconds:
         0.081  \set aid random(1, 100000 * :scale)
        10.140  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```



**读写测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 53323
latency average = 180.360 ms
latency stddev = 362.124 ms
tps = 441.405058 (including connections establishing)
tps = 441.995753 (excluding connections establishing)
statement latencies in milliseconds:
         0.015  \set aid random(1, 100000 * :scale)
         0.013  \set bid random(1, 1 * :scale)
         0.010  \set tid random(1, 10 * :scale)
         0.011  \set delta random(-5000, 5000)
         1.495  BEGIN;
         9.882  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         3.228  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         6.328  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        21.077  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         3.376  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       134.923  END;
```



### 分布式表(9个shard)测试

将上诉的表删掉

```
drop table pgbench_accounts; 
drop table pgbench_branches;
drop table pgbench_history; 
drop table pgbench_tellers;
```

重新生成三千万条数据

```
/usr/pgsql-12/bin/pgbench -i -s300 -h 10.10.10.140 -p 5432 -U postgres -d postgres
```

创建分布式表

```plsql
--将tpc-b涉及的4张表转换为sharding表（9个shard,每台物理机每个PG实例3个shard）
set citus.shard_count=9;
select create_distributed_table('pgbench_accounts','aid');
select create_distributed_table('pgbench_branches','bid');  
select create_distributed_table('pgbench_tellers','tid');  
select create_distributed_table('pgbench_history','aid');  

-- 删除本地数据

SELECT truncate_local_data_after_distributing_table($$public.pgbench_accounts$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_branches$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_tellers$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_history$$);
```



**只读测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
transaction type: <builtin: select only>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 1143499
latency average = 8.274 ms
latency stddev = 12.815 ms
tps = 9518.062828 (including connections establishing)
tps = 9524.721460 (excluding connections establishing)
statement latencies in milliseconds:
         0.103  \set aid random(1, 100000 * :scale)
         8.166  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

**读写测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
 transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 70721
latency average = 135.808 ms
latency stddev = 287.152 ms
tps = 587.663842 (including connections establishing)
tps = 588.195306 (excluding connections establishing)
statement latencies in milliseconds:
         0.014  \set aid random(1, 100000 * :scale)
         0.012  \set bid random(1, 1 * :scale)
         0.013  \set tid random(1, 10 * :scale)
         0.013  \set delta random(-5000, 5000)
         1.380  BEGIN;
         4.545  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         3.157  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         5.875  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        17.250  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         3.345  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       100.207  END;
```



## 压测环境2

集群配置改成为 `4C8G`

| hostname |           ip | os         | 配置  |
| -------- | -----------: | ---------- | ----- |
| test     | 10.10.10.138 | CentOS 7.8 | 8C16G |

**主要软件**

- CentOS 7.8
- PostgreSQL `12.6`

### 本地表测试

生成三千万条数据

```
/usr/pgsql-12/bin/pgbench -i -s300 -h 10.10.10.140 -p 5432 -U postgres -d postgres
```

**只读测试**

```
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres

...
transaction type: <builtin: select only>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 1127843
latency average = 5.542 ms
latency stddev = 12.636 ms
tps = 9390.602342 (including connections establishing)
tps = 9392.982958 (excluding connections establishing)
statement latencies in milliseconds:
         2.697  \set aid random(1, 100000 * :scale)
         2.845  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

**读写测试**

```bash
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres

...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 62798
latency average = 153.551 ms
latency stddev = 901.965 ms
tps = 516.127084 (including connections establishing)
tps = 516.335863 (excluding connections establishing)
statement latencies in milliseconds:
         1.196  \set aid random(1, 100000 * :scale)
         1.112  \set bid random(1, 1 * :scale)
         1.013  \set tid random(1, 10 * :scale)
         0.981  \set delta random(-5000, 5000)
         1.767  BEGIN;
         3.251  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         2.492  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         4.839  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        19.503  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         2.474  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       114.947  END;
```



###  分布式表（3个shard）测试



创建分布式表，使用上诉表

```plsql
--将tpc-b涉及的4张表转换为sharding表（3个shard,每台物理机每个PG实例1个shard）
set citus.shard_count=3;
select create_distributed_table('pgbench_accounts','aid');
select create_distributed_table('pgbench_branches','bid');  
select create_distributed_table('pgbench_tellers','tid');  
select create_distributed_table('pgbench_history','aid');  

-- 删除本地数据

SELECT truncate_local_data_after_distributing_table($$public.pgbench_accounts$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_branches$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_tellers$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_history$$);
```



**只读测试**

```
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres

transaction type: <builtin: select only>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 1320148
latency average = 6.630 ms
latency stddev = 11.898 ms
tps = 10994.656166 (including connections establishing)
tps = 11000.012305 (excluding connections establishing)
statement latencies in milliseconds:
         0.659  \set aid random(1, 100000 * :scale)
         5.971  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```



**读写测试**

```
/usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 52340
latency average = 184.292 ms
latency stddev = 427.537 ms
tps = 430.793731 (including connections establishing)
tps = 431.012090 (excluding connections establishing)
statement latencies in milliseconds:
         0.076  \set aid random(1, 100000 * :scale)
         0.070  \set bid random(1, 1 * :scale)
         0.057  \set tid random(1, 10 * :scale)
         0.052  \set delta random(-5000, 5000)
         1.201  BEGIN;
         3.575  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         2.414  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         5.885  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        21.013  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         2.671  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       147.343  END;
```



### 分布式表(9个shard)测试

将上诉的表删掉

```
drop table pgbench_accounts; 
drop table pgbench_branches;
drop table pgbench_history; 
drop table pgbench_tellers;
```

重新生成三千万条数据

```
/usr/pgsql-12/bin/pgbench -i -s300 -h 10.10.10.140 -p 5432 -U postgres -d postgres
```

创建分布式表

```plsql
--将tpc-b涉及的4张表转换为sharding表（9个shard,每台物理机每个PG实例3个shard）
set citus.shard_count=9;
select create_distributed_table('pgbench_accounts','aid');
select create_distributed_table('pgbench_branches','bid');  
select create_distributed_table('pgbench_tellers','tid');  
select create_distributed_table('pgbench_history','aid');  

-- 删除本地数据

SELECT truncate_local_data_after_distributing_table($$public.pgbench_accounts$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_branches$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_tellers$$);
SELECT truncate_local_data_after_distributing_table($$public.pgbench_history$$);
```



**只读测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -S -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
transaction type: <builtin: select only>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 990600
latency average = 9.030 ms
latency stddev = 15.885 ms
tps = 8251.793713 (including connections establishing)
tps = 8256.988199 (excluding connections establishing)
statement latencies in milliseconds:
         0.661  \set aid random(1, 100000 * :scale)
         8.368  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
```

**读写测试**

```
 /usr/pgsql-12/bin/pgbench -M prepared -v -r -P1 -c80 -j80 -T 120 -h 10.10.10.140 -p 5432 -U postgres -d postgres
 
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 300
query mode: prepared
number of clients: 80
number of threads: 80
duration: 120 s
number of transactions actually processed: 25285
latency average = 380.426 ms
latency stddev = 732.740 ms
tps = 209.508321 (including connections establishing)
tps = 209.584737 (excluding connections establishing)
statement latencies in milliseconds:
         0.073  \set aid random(1, 100000 * :scale)
         0.059  \set bid random(1, 1 * :scale)
         0.061  \set tid random(1, 10 * :scale)
         0.057  \set delta random(-5000, 5000)
         1.105  BEGIN;
         2.780  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         1.604  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         6.957  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        44.016  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         1.651  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
       321.968  END;
```



## 数据汇总

| 环境（2C4G）       | test case         | TPS         | 平均延迟   |
| :----------------- | :---------------- | :---------- | ---------- |
| local PG           | 三千万 tpc-b 只读 | 3195.576593 | 24.984 ms  |
| citus(1+3) shard=3 | 三千万 tpc-b 只读 | 7734.042196 | 10.225 ms  |
| citus(1+3) shard=9 | 三千万 tpc-b 只读 | 9518.062828 | 8.274 ms   |
| local PG           | 三千万 tpc-b 读写 | 316.864187  | 251.683 ms |
| citus(1+3) shard=3 | 三千万 tpc-b 读写 | 441.405058  | 180.360 ms |
| citus(1+3) shard=9 | 三千万 tpc-b 读写 | 587.663842  | 135.808 ms |



| 环境（4C8G）       | test case         | TPS          | 平均延迟   |
| :----------------- | :---------------- | :----------- | ---------- |
| local PG           | 三千万 tpc-b 只读 | 9390.602342  | 5.542 ms   |
| citus(1+3) shard=3 | 三千万 tpc-b 只读 | 10994.656166 | 6.630 ms   |
| citus(1+3) shard=9 | 三千万 tpc-b 只读 | 8251.793713  | 9.030 ms   |
| local PG           | 三千万 tpc-b 读写 | 516.127084   | 153.551 ms |
| citus(1+3) shard=3 | 三千万 tpc-b 读写 | 430.793731   | 184.292 ms |
| citus(1+3) shard=9 | 三千万 tpc-b 读写 | 209.508321   | 380.426 ms |

