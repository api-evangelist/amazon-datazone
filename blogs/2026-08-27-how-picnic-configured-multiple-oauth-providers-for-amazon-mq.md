---
title: "How Picnic configured multiple OAuth providers for Amazon MQ"
url: "https://aws.amazon.com/blogs/big-data/how-picnic-configured-multiple-oauth-providers-for-amazon-mq/"
date: "2026-08-27"
author: "Oscar Mapfumo Sibanda"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
Picnic runs RabbitMQ as the messaging backbone for hundreds of microservices on Amazon MQ for RabbitMQ. This post shows how to configure one broker to trust multiple OAuth 2.0 identity providers, Keycloak for operators and AWS IAM for services, so you can eliminate static credentials while maintaining separate identity paths for people and workloads.
