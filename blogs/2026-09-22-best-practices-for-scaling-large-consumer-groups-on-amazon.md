---
title: "Best practices for scaling large consumer groups on Amazon MSK"
url: "https://aws.amazon.com/blogs/big-data/best-practices-for-scaling-large-consumer-groups-on-amazon-msk/"
date: "2026-09-22"
author: "Pallavi Jha"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
As consumer groups on Amazon MSK scale to thousands of members, the metadata record Kafka writes during rebalances can exceed the 1 MB limit and stall the group. Learn how to estimate metadata size, raise the topic-level limit safely, plan capacity, and apply complementary strategies for scaling large consumer groups.
