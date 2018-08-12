# Postgres Test Summary

## Experiment Overview
The results in standalone tests shows the average performance of current implementation of ```pg_qsort``` is quite impressive, especially on paritial sorted and low cardinality data. (the details can be found in ```standalone_test.md```)

However, the test results also shows that ```pg_qsort``` can have extremely bad performance on the killer sequence, which is a specially generated data pattern to push ```pg_qsort``` to the worst case scenario. This **might** cause some security issues in PostgreSQL, though it's still debatable.

That's where an intro sort implementation of ```pg_qsort``` come to place. In the standalone test, we found intro sort has ```O(nlogn)``` worst case time complexity while losing little on average cases. Therefore, an intro sort implemenation may be worth a try.

Comparing two sorting routines in PostgreSQL may be tricky. The original approach that I was using can be found [here](https://github.com/Strider-Alex/GSoC-2018-archive/blob/master/reports/new_qsort_report.pdf) (you can also find some statistics about how killer sequence influences the performance of ```select``` queries in PostgreSQL). However, there'are some better scripts kindly provided by @tvondra to test the performance, which produce more reliable and detailed results. 

Bascially, a series of ```select``` and ```create index``` queries are used to measure the performance of the sorting routine. The variables are:
- data size
- data type
- data pattern
- cardinality
- work memory size
- number of workers

The tests are carried out in **two** environments. One is on the (quite old) i5-2500k CPU, anotehr is on the (much newer) e5-2620v4 system. 

## Experiment Steps
1. Get ```run-bench.sh```, ```sort-bench.sh``` and ```master.conf``` under this [repo](https://bitbucket.org/tvondra/sort-intro-sort-i5-2/src)
2. Install the master branch of PostgreSQL to ```/var/lib/postgresql/pg-master/```
3. Apply a sorting patch and install to ```/var/lib/postgresql/pg-patched/```
4. Run ```run-bench.sh``` and wait for the results to be generated.
5. Repeat 2-4 for each sorting patch we want to test.

## Experiment Results

**Notice**: the detailed results can be found under ```analyzed_data```

The first tested implemention of intro sort does only one presorted check on the whole array (original ```pg_qsort``` does presorted check in every recursion).

### i5-2500K

Values smaller than 100% indicates a performance gain.

N = 10K

| Data | #worker | Create Index | Select Query |
| --- | --- |---| ---|
| high_cardinality_almost_asc| 0 | 100.07% | 99.19% |
| high_cardinality_almost_asc| 4 | 101.67% | 99.21% |
| high_cardinality_random| 0 | 101.22% | 99.02% |
| high_cardinality_random| 4 | 100.95% | 98.95% |
| high_cardinality_sorted| 0 | 99.44% | 98.98% |
| high_cardinality_sorted| 4 | 100.43% | 99.07% |
| low_cardinality_almost_asc| 0 | 113.01% | 100.44% |
| low_cardinality_almost_asc| 4 | 113.86% | 100.38% |
| low_cardinality_random| 0 | 100.36% | 98.10% |
| low_cardinality_random| 4 | 100.33% | 98.21% |
| low_cardinality_sorted| 0 | 100.36% | 98.10% |
| low_cardinality_sorted| 4 | 100.33% | 98.21% |

The results shows the sorting patch brings a little gain on ```select``` queries and a little regression on ```create index``` queries, except for the low cardinality almost ascendent data, where we can observe a regression of about 10% on ```create index``` queries.

For larger data set (N = 100K and 1M), the results are  very similar. For N = 100K, this regression can reach about 30%. And for N = 1M, it drops back to about 6%.

The orignal test also tests unique data, but the results are very similar with high cardinality data.

### e5-2620v4

N = 10K

| Data | #worker | Create Index | Select Query |
| --- | --- |---| ---|
| high_cardinality_almost_asc| 0 | 104.08% | 97.50% |
| high_cardinality_almost_asc| 4 | 105.07% | 98.21% |
| high_cardinality_random| 0 | 103.88% | 97.61% |
| high_cardinality_random| 4 | 105.88% | 97.74% |
| high_cardinality_sorted| 0 | 100.38% | 97.69% |
| high_cardinality_sorted| 4 | 102.07% | 97.72% |
| low_cardinality_almost_asc| 0 | 127.36% | 98.49% |
| low_cardinality_almost_asc| 4 | 125.42% | 98.97% |
| low_cardinality_random| 0 | 103.85% | 96.62% |
| low_cardinality_random| 4 | 101.61% | 96.36% |
| low_cardinality_sorted| 0 | 99.72% | 98.37% |
| low_cardinality_sorted| 4 | 100.84% | 98.51% |

The regression on low cardinality almost ascendent data is more dramatic on the faster CPU, where we can observe a regression of about 25% on ```create index``` queries. For N = 100K, there is a 35% regression for ```create index``` queries on low cardinality ascendent data and 10% on other data. For N = 1M, the regression on low cardinality ascendent data drops to about 8%. On other data there's a 3%-5% peformance gain.

### Variation of Intro Sort on i5-2500K
Previous tests show that the new intro sort implementation has large regression on low cardinality almost ascendent data, which is unlikely to be accepted. Therefore, we decide to bring back the presorted check in ```pg_qsort``` sort to every recursion of intro sort instead of only checking once on the whole array. Then we rerun test on the i5 CPU.

N = 10K

| Data | #worker | Create Index | Select Query |
| --- | --- |---| ---|
| high_cardinality_almost_asc| 0 | 100.89% | 100.61% |
| high_cardinality_almost_asc| 4 | 100.32% | 100.74% |
| high_cardinality_random| 0 | 101.90% | 100.09% |
| high_cardinality_random| 4 | 101.74% | 100.18% |
| high_cardinality_sorted| 0 | 100.71% | 100.89% |
| high_cardinality_sorted| 4 | 100.42% | 100.91% |
| low_cardinality_almost_asc| 0 | 100.62% | 100.55% |
| low_cardinality_almost_asc| 4 | 100.17% | 100.58% |
| low_cardinality_random| 0 | 99.35% | 100.13% |
| low_cardinality_random| 4 | 98.97% | 99.79% |
| low_cardinality_sorted| 0 | 96.92% | 100.79% |
| low_cardinality_sorted| 4 | 97.11% | 100.83% |

There are nearly no difference between the performance of these two sorting routines (at least not observable in PostgreSQL). For N = 100K, there's an about 2% regression for ```create index``` queries and 3% gain for ```select``` queries. For N = 1M, the performance is nearly the same.

This new version of intro sort may worth a try because it provides a guaranteed ```O(nlogn)``` without much loss on average performance. 

## Limitations
Testing sorting routines in PostgreSQL is much more time intensivethe than in a standalone environment. To shorten the runtime, each test case only runs 5 times to get the average value. This might not be enough. In ```line 304``` of ```result_i5.ods``` (n=10000), for instance, we can see the performance comparison suddenly jumps to 185.93%, and immediately falls to 60.42%
afterwards. In fact, these values should be close to 100% because the what two sorting routines do is just a linear presorted check and nothing else.

Moreover, we still haven't tested the new version of introsort on the faster CPU yet, for which the result may be quite different. 
