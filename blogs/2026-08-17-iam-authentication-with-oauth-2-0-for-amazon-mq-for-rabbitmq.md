---
title: "IAM authentication with OAuth 2.0 for Amazon MQ for RabbitMQ"
url: "https://aws.amazon.com/blogs/big-data/iam-authentication-with-oauth-2-0-for-amazon-mq-for-rabbitmq/"
date: "2026-08-17"
author: "Vinodh Kannan Sadayamuthu"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
IAM authentication with OAuth 2.0 lets clients connect to Amazon MQ for RabbitMQ using their existing IAM identity instead of static broker-local credentials. This post covers the key rabbitmq.conf configuration for using AWS IAM as an OAuth 2.0 provider and shows a multi-tenant example with vhost isolation enforced by IAM roles and broker scope aliases.
