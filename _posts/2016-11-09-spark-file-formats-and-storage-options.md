---
layout: post
title:  "Spark - File formats and storage options"
date:   2016-11-08 11:00:00
tags: spark
language: EN
---

I recently worked on a project in which Spark was used to ingest data from text files. The team was struggling to read the files with acceptable performance. That lead me to analysing a few options that are offered to you when using Spark. I am reviewing them here.


# Some context

The data files are text files with one record per line in a custom format. Files contain from 10 to 40 million records. In the tests that follow, I used a 14.4 GB file containing 40 million records.

The files are received from an external system, meaning we can ask to be sent a compressed file but not more complex formats (Parquet, Avro...).

Finally, Spark is used on a standalone cluster (i.e. not on top of Hadoop) on Amazon AWS. Amazon S3 is usually used to store files.


# Options to be tested

I tested multiple combinations:

- Either a plain or a compressed file. I tested 2 compression formats: GZ (very common, fast, but not splittable) and BZ2 (splittable but very CPU expensive).
- Reading from an EBS drive or from S3. S3 is what is used in real life but the disk serves as a baseline to assess the performance of S3.
- With or without repartitioning. In a real use case, repartitioning is mandatory to achieve good parallelism when the initial partitioning is not adequate. This does not apply to uncompressed files as they already generate enough partitions. When repartitioning, I asked for 460 partitions as this is the number of partitions created when reading the uncompressed file (14.4 GB / 32 MB).
- Spark versions 2.0.1 vs Spark 1.6.0. I tested the version currently used by the client (1.6.0) and the latest version available from Apache (2.0.1).


# The test

The test I ran is very simple. It simply reads the file and counts the number of lines. This is really all we need to assess the performance of reading the file.

The code I wrote only leverages Spark RDDs to focus on read performance:

```scala
val filename = "<path to the file>"
val file = sc.textFile(filename)
file.count()
```

In the measures below, when the test says "Read + repartition", the file is repartitioned before counting the lines. Repartitioning is required in real-life applications when the initial number of partitions is too low. This ensures that all the cores available on the cluster are used. The code used in this case is the following:

```scala
val filename = "<path to the file>"
val file = sc.textFile(filename).reparition(460)
file.count()
```

A few additional details:

- Tests are run on a Spark cluster with 3 *c4.4xlarge* workers (16 vCPUs and 30 GB of memory each).
- Code is run in a *spark-shell*.
- The number of partitions and the time taken to read the file are read from the Spark UI.
- When files are read from S3, the *S3a* protocol is used.


# Measures

With Spark 2.0.1:

| Format | File size | Source | Test | Time | # partitions |
| ------ | --------- | ------ | ---- | ---- | ------------ |
| Uncompressed | 14.4 GB | EBS | Read | 3 s | 460 |
| Uncompressed | 14.4 GB | S3 | Read | 13 s | 460 |
| GZ | 419.3 MB | EBS | Read | 47 s | 1 |
| GZ | 419.3 MB | EBS | Read + repartition | 1.5 min | 460 |
| GZ | 419.3 MB | S3 | Read | 44 s | 1 |
| GZ | 419.3 MB | S3 | Read + repartition | 1.4 min | 460 |
| BZ2 | 236.3 MB | EBS | Read | 55 s | 8 |
| BZ2 | 236.3 MB | EBS | Read + repartition | 1.2 min | 460 |
| BZ2 | 236.3 MB | S3 | Read | 1.1 min | 8 |
| BZ2 | 236.3 MB | S3 | Read + repartition | 1.5 min | 460 |

With Spark 1.6.0:

| Format | File size | Source | Test | Time | # partitions |
| ------ | --------- | ------ | ---- | ---- | ------------ |
| Uncompressed | 14.4 GB | EBS | Read | 3 s | 460 |
| Uncompressed | 14.4 GB | S3 | Read | 19 s | 460 |
| GZ | 419.3 MB | EBS | Read | 44 s | 1 |
| GZ | 419.3 MB | EBS | Read + repartition | 1.7 min | 460 |
| GZ | 419.3 MB | S3 | Read | 46 s | 1 |
| GZ | 419.3 MB | S3 | Read + repartition | 1.8 min | 460 |
| BZ2 | 236.3 MB | EBS | Read | 53 s | 8 |
| BZ2 | 236.3 MB | EBS | Read + repartition | 1.2 min | 460 |
| BZ2 | 236.3 MB | S3 | Read | 57 s | 8 |
| BZ2 | 236.3 MB | S3 | Read + repartition | 1.1 min | 460 |


# Conclusions

**Spark version** - Measures are very similar between Spark 1.6 and Spark 2.0. This makes sense as this test uses plain RDDs (Catalyst or Tungsten cannot perform any optimization).

**EBS vs S3** - S3 is slower than the EBS drive (clearly seen when reading uncompressed files). Performance of S3 is still very good, though, with a combined throughput of 1.1 GB/s. Also, keep in mind that EBS drives have drawbacks: files are not shared between servers (they have to be replicated manually) and IOPS can be throttled.

**Compression** - GZ files are not ideal because they are not splittable and therefore require repartitioning. BZ2 files suffer from a similar problem: although they are splittable, they are so much compressed that you get very few partitions (8, in this case, on a cluster with 48 cores). The other problem is that the performance of BZ2 files is poor compared to uncompressed files. In the end, we see that uncompressed files clearly outperform compressed files. This is because uncompressed files are I/O bound and compressed files are CPU bound, but I/Os are good enough here.


## My recommendation

Given this, **I recommend storing files on S3 as uncompressed files**. This allows to achieve great performance while providing a safe storage. Keep in mind that this recommendation only applies when you don't have control over the input format. When you do, a more structured format (e.g. Parquet) would be more appropriate, especially since they also offer powerful capabilities (columnar storage, fine-grained compression, etc.).

Final note about GZ files: they are actually to be avoided because they will pull the whole file in a single partition in the driver of your application, meaning you can easily get an *Out of memory* error.
