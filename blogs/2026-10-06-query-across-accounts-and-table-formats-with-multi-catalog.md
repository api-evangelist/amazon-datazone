---
title: "Query across accounts and table formats with multi-catalog in Amazon EMR"
url: "https://aws.amazon.com/blogs/big-data/query-across-accounts-and-table-formats-with-multi-catalog-in-amazon-emr/"
date: "2026-10-06"
author: "Suthan Phillips"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Amazon EMR 8.1.0 adds multi-catalog support through the RedirectingSessionCatalog, so you can query Iceberg, Delta Lake, Hudi, and Hive tables through a single catalog and join data across AWS accounts without copying it. This post shows how to put these capabilities into practice on Amazon EMR Serverless.
