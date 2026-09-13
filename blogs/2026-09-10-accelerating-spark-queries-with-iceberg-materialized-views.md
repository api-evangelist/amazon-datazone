---
title: "Accelerating Spark queries with Iceberg materialized views"
url: "https://aws.amazon.com/blogs/big-data/accelerating-spark-queries-with-iceberg-materialized-views/"
date: "2026-09-10"
author: "Yuzhou Sun"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Accelerate slow, repetitive Apache Spark analytical queries on Apache Iceberg tables without rewriting any SQL. This post shows how automatic query rewrite in Amazon EMR and AWS Glue uses Iceberg materialized views in the AWS Glue Data Catalog to transparently substitute matching query plans, and how to design materialized views for the best speedup.
