---
title: "Enable cross-cloud analytics with Amazon S3 Tables and Google BigQuery, Part 1: IAM-based access control"
url: "https://aws.amazon.com/blogs/big-data/enable-cross-cloud-analytics-with-amazon-s3-tables-and-google-bigquery-part-1-iam-based-access-control/"
date: "2026-08-25"
author: "Lakshmi Nair"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Your Google BigQuery users need to query data that lives in Amazon S3 Tables on AWS without copying it across clouds. This post shows how to connect BigQuery to Amazon S3 Tables through the AWS Glue Iceberg REST Catalog using IAM-based access control, so you keep one governed dataset and query it live from BigQuery.
