---
title: "How Autodesk migrated 2.3 billion documents to Amazon OpenSearch Service using Migration Assistant and intelligent routing"
url: "https://aws.amazon.com/blogs/big-data/how-autodesk-migrated-2-3-billion-documents-to-amazon-opensearch-service-using-migration-assistant-and-intelligent-routing/"
date: "2026-08-14"
author: "Ambarish Rao"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
This post walks through how Autodesk re-architected a single-index Elasticsearch 7.1.1 domain on Amazon OpenSearch Service into four multi-index OpenSearch Service domains, using Migration Assistant for Amazon OpenSearch Service and a routing layer that directs each query to the shards that hold the data for that query.
