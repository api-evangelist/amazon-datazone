---
title: "How to build a cross-Region resilience for Amazon OpenSearch Service with Amazon MSK"
url: "https://aws.amazon.com/blogs/big-data/how-to-build-a-cross-region-resilience-for-amazon-opensearch-service-with-amazon-msk/"
date: "Mon, 11 May 2026 18:46:43 +0000"
author: "Sriharsha Subramanya Begolli"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
In this post, we outline the solution that provides cross-Region resiliency without needing to reestablish relationships during a fail-back, using an active-active replication model with Amazon OpenSearch Ingestion (OSI) and Amazon Managed Streaming for Apache Kafka (Amazon MSK). This solution applies to both OpenSearch Service managed clusters and Amazon OpenSearch Serverless collections. We use Amazon OpenSearch Serverless as an example for the configurations in this post.
