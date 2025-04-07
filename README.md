# Making Postgres better with OrioleDB

One of the biggest advantages Postgres has over lots of other DBMSs out there is the fact that it allows external improvements or new features, without requiring any interactions with the core codebase, via extensions. There are many extensions each with a unique purpose(I have also written two trivial extensions [pg_wal_ext](https://github.com/misachi/pg_wal_ext) and [pg_table_bloat](https://github.com/misachi/pg_table_bloat) )

One extension that especially stands out is [OrioleDB](https://www.orioledb.com/). It provides an alternative storage engine to  Postgres. Postgres comes with only one storage engine based on the heap. This is different from MySQL which comes with several storage engines: innodb, myissam etc.

Postgres heap based storage engine works well in most cases but it also presents a number of issues, some of which are: [bloat introduced by how updates are handled](https://www.cs.cmu.edu/~pavlo/blog/2023/04/the-part-of-postgresql-we-hate-the-most.html), [the need for garbage collection(vacuum)](https://www.postgresql.org/docs/17/sql-vacuum.html), [transaction wraparound](https://www.cybertec-postgresql.com/en/transaction-id-wraparound-a-walk-on-the-wild-side/)

OrioleDB comes with the promise of solving the issues presented by the Postgres heap. I tested it out and the results were pretty good. I'll describe the process I used and the results below.

First you need a patched up version of Postgres: [16(tag:patches16_33)](https://github.com/orioledb/postgres/archive/refs/tags/patches16_33.tar.gz) or [17(tag:patches17_5)](https://github.com/orioledb/postgres/archive/refs/tags/patches17_5.tar.gz). The steps are available on [github](https://github.com/orioledb/orioledb?tab=readme-ov-file#build-from-source)

The test is for a read only workload on 16 tables each with 25M records(approx 94GB for each setup). I used a server with 64GB RAM, 20cores and 500GB NVME SSD(ext4 filesystem). I use my setup [scripts](https://github.com/misachi/postgres-scripts) to build and install Postgres. The benchmarking tool used is [sysbench](https://github.com/akopytov/sysbench?tab=readme-ov-file#building-and-installing-from-source). The OrioleDB used is built from source from the main branch commit `0c484c4`.

The install steps for OrioleDB:
```
# Install dependencies
apt update
apt install python3 python3-dev python3-pip python3-setuptools python3-testresources libzstd1 libzstd-dev libssl-dev libcurl4-openssl-dev

git clone https://github.com/orioledb/orioledb
cd orioledb
git reset --hard 0c484c4 # optional
make USE_PGXS=1 ORIOLEDB_PATCHSET_VERSION=5
echo "shared_preload_libraries = 'orioledb.so'" >> /usr/local/pgsql/data/postgresql.conf
```

The buffer pool for both setups is set to 16GB i.e `shared_buffers` for heap and `orioledb.main_buffers` for Orioledb.

To run the actual tests, I used https://github.com/misachi/sysbench-graphing-tests. Once everything has been setup, running `./run.threads <run number> pgsql` should execute the tests. The script ensures the database is warmed up before running the tests.

I did 2 test runs for each setup: 2 runs of Postgres heap and 2 runs of Postgres with OrioleDB extension. The results are as shown below

![result](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8pynmoo67y2h0i5ois15.png)

Runs labelled `out/res2` and `out/res3` show results for running normal Postgres with heap based storage engine while `res4` and `res5` show results for Postgres with OrioleDB.

The impressive bit, OrioleDB is able to outperform Postgres heap while using less CPU and memory resources. This can be attributed partly to OrioleDB's lock-free page reads thus reduced contention and index-organized tables.

Memory usage(MB) for Postgres heap

![heap](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vs2zkbrur47doz1v4rom.png)

CPU usage for Postgres heap

![heap](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ao426cjbdg4mvwvkhwm6.png)

Memory usage(MB) for OrioleDB

![Orioledb](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ydwjnukzvlo7r5i3r4vi.png)

CPU usage for OrioleDB

![Orioledb](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jqyse13lpg312l4mxijv.png)

Originally posted [here](https://dev.to/misachi/making-postgres-better-with-orioledb-49cp)