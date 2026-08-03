---
title: "Lowering AWS KMS decrypt API costs in EMR Spark jobs"
url: "https://aws.amazon.com/blogs/big-data/lowering-aws-kms-decrypt-api-costs-in-emr-spark-jobs/"
date: "2026-07-30"
author: "Navaneedha Krishnan Jagathesan"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Processing encrypted data in Amazon S3 with Amazon EMR and Apache Spark can drive up AWS KMS decrypt API costs as the number of objects grows. This post shows three techniques to reduce those costs without compromising encryption: optimizing file formats (including Apache Iceberg), aggregating data, and using AWS Glue Data Catalog partition indexes.
