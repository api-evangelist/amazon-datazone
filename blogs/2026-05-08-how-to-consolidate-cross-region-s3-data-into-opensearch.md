---
title: "How to consolidate cross-Region S3 data into OpenSearch"
url: "https://aws.amazon.com/blogs/big-data/how-to-consolidate-cross-region-s3-data-into-opensearch/"
date: "2026-05-08"
author: "David Venable"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
You might have data in Amazon Simple Storage Service (Amazon S3) buckets in different AWS Regions that you want available in a single Amazon OpenSearch Service domain or collection. Consolidating data across Regions provides unified analytics and searches, reduce operation complexity, and streamline your search infrastructure. We’re happy to announce that Amazon OpenSearch Ingestion pipelines can now read from S3 buckets in different Regions to ingest and consolidate data into a single OpenSearch Service domain or collection.
