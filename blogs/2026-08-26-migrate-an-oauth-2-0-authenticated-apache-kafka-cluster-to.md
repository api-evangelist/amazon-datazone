---
title: "Migrate an OAuth 2.0 authenticated Apache Kafka cluster to Amazon MSK with MSK Replicator"
url: "https://aws.amazon.com/blogs/big-data/migrate-an-oauth-2-0-authenticated-apache-kafka-cluster-to-amazon-msk-with-msk-replicator/"
date: "2026-08-26"
author: "Subham Rakshit"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
MSK Replicator now supports OAuth 2.0 (SASL/OAUTHBEARER) authentication to external Apache Kafka clusters. This post walks through the three supported grant types, how to configure Replicator for each, the network and TLS prerequisites that are commonly missed, and how to handle identity providers behind an additional federation layer.
