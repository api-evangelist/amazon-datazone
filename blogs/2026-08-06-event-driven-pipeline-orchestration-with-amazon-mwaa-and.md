---
title: "Event-driven pipeline orchestration with Amazon MWAA and Airflow 3.0"
url: "https://aws.amazon.com/blogs/big-data/event-driven-pipeline-orchestration-with-amazon-mwaa-and-airflow-3-0/"
date: "2026-08-06"
author: "Satya Chikkala"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Data engineering teams running Apache Airflow across multiple AWS accounts have no built-in way to coordinate workflows between separate Amazon MWAA environments. With Airflow 3.0 on Amazon MWAA, you can use asset-based scheduling and Asset Watchers with Amazon SQS to build event-driven, cross-account orchestration that replaces polling with near real-time triggers.
