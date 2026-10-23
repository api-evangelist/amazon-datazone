---
title: "Resolving concurrent write conflicts for Apache Iceberg tables in Amazon Redshift"
url: "https://aws.amazon.com/blogs/big-data/resolving-concurrent-write-conflicts-for-apache-iceberg-tables-in-amazon-redshift/"
date: "2026-10-13"
author: "Sanjay Devireddy"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Amazon Redshift now automatically resolves many concurrent writes to Apache Iceberg tables that don't actually overlap, committing changes that previously failed and had to retry. Learn why these writes conflict, how the engine rebases non-overlapping ones across operation, partition, and file checks, and the throughput gains it delivers.
