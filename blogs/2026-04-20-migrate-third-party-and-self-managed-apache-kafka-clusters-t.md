---
title: "Migrate third-party and self-managed Apache Kafka clusters to Amazon MSK Express brokers with Amazon MSK Replicator"
url: "https://aws.amazon.com/blogs/big-data/migrate-third-party-and-self-managed-apache-kafka-clusters-to-amazon-msk-express-brokers-with-amazon-msk-replicator/"
date: "Mon, 20 Apr 2026 20:00:27 +0000"
author: "Ankita Mishra"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Migrating Apache Kafka workloads to the cloud often involves managing complex replication infrastructure, coordinating application cutovers with extended downtime windows, and maintaining deep expertise in open-source tools like Apache Kafka’s MirrorMaker 2 (MM2). These challenges slow down migrations and increase operational risk. <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator.html" rel="noopener noreferrer" target="_blank">Amazon MSK Replicator</a> addresses these challenges, enabling you to migrate your Kafka deployments (referred to as “external” Kafka clusters) to <a href="https://aws.amazon.com/msk/" rel="noopener noreferrer" target="_blank">Amazon MSK</a> <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-broker-types-express.html" rel="noopener noreferrer" target="_blank">Express brokers</a> with minimal operational overhead and reduced downtime. MSK Replicator supports data migration from Kafka deployments (version 2.8.1 or later) that have <a href="https://kafka.apache.org/42/security/authentication-using-sasl/" rel="noopener noreferrer" target="_blank">SASL/SCRAM authentication</a> enabled – including Kafka clusters running on-premises, on AWS, or other cloud providers, as well as Kafka-protocol-compatible services like Confluent Platform, Avien, RedPanda, WarpStream, or AutoMQ when configured with SASL/SCRAM authentication.</p> 
<p>In this post, we walk you through how to replicate Apache Kafka data from your external Apache Kafka deployments to Amazon MSK Express brokers using MSK Replicator. You will learn how to configure authentication on your external cluster, establish network connectivity, set up bidirectional replication, and monitor replication health to achieve a low-downtime migration.</p> 
<h2>How it works</h2> 
<p>MSK Replicator is a fully managed serverless service that replicates topics, configurations, and offsets from cluster to cluster. It alleviates the need to manage complex infrastructure or configure open-source tools.</p> 
<p><img alt="" class="alignnone size-full wp-image-90053" height="3044" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5876-1.png" width="3084" /></p> 
<p>Before MSK Replicator, customers used tools like MM2 for migrations. These tools lack bi-directional topic replication when using the same topic names, creating complex application architectures to consume different topics on different clusters. Custom replication policies in MM2 can allow identical topic names, but MM2 still lacks bidirectional offset replication because the MM2 architecture requires producers and consumers to run on the same cluster to replicate offsets. This created complex migrations that required either migrating consumers before producers or big-bang migrations migrating all applications at once. When customers run into issues during the migration, the rollback process is error-prone and introduces large amounts of duplicate message processing due to the lack of consumer group offset synchronization. These approaches create risk and complexity for customers that make migrations difficult to manage.</p> 
<p>MSK Replicator addresses these problems by supporting bidirectional replication of data and enhanced consumer group offset synchronization. MSK Replicator copies topics and offsets from an external Kafka cluster to MSK, allowing you to preserve the same topic and consumer group names on both clusters. MSK Replicator also supports creating a second Replicator instance for bidirectional replication of both data and enhanced offset synchronization, allowing producers and consumers to run independently on different Kafka clusters. Data published or consumed on the Amazon MSK cluster will be replicated back to the external cluster by the second Replicator. This feature works when producers and consumers are migrated regardless of order without worrying about dependencies between applications.</p> 
<p>Because MSK Replicator provides bidirectional data replication and enhanced consumer group offset synchronization, you can move producers and consumers at your own pace without data loss. This reduces migration complexity, allowing you to migrate applications between your external Kafka cluster and Amazon MSK regardless of order. If you run into problems during the migration, enhanced offset synchronization allows you to roll back changes by moving applications back to the external Kafka cluster, where they restart from the latest checkpoint from the Amazon MSK cluster.</p> 
<p>For example, consider three applications:</p> 
<ol> 
 <li>The “Orders” application, which accepts incoming orders and writes them to the orders Kafka topic</li> 
 <li>The “Order status” application, which reads from the “orders” Kafka topic and writes status updates to the <code>order_status</code> topic</li> 
 <li>The “Customer notification” application, which reads from the <code>order_status</code> topic and notifies customers when status changes</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-90054" height="1364" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/BDB-5876-2.png" width="3764" /></p> 
<p>MSK Replicator enables these applications to be migrated between an on-premises Apache Kafka cluster and an Amazon MSK Express cluster with low downtime and no data loss, regardless of order. The “Order status” application can migrate first, receive orders from the on-premises “Orders” application, and send status updates to the on-premises “Customer notification” application. If issues arise during the migration, the “Order status” application can roll back to the on-premises cluster and its consumer group offsets for the orders topic will be ready for it to pick up from where it left off on the Amazon MSK cluster.</p> 
<p>MSK Replicator supports data distribution across hybrid and multi-cloud environments for analytics, compliance, and business continuity. It is also configured for disaster recovery scenarios where Amazon MSK Express serves as a resilient target for your external Kafka clusters.</p> 
<p>If you are currently using MM2 for replication, see <a href="https://aws.amazon.com/blogs/big-data/amazon-msk-replicator-and-mirrormaker2-choosing-the-right-replication-strategy-for-apache-kafka-disaster-recovery-and-migrations/" rel="noopener noreferrer" target="_blank">Amazon MSK Replicator and MirrorMaker2: Choosing the right replication strategy for Apache Kafka disaster recovery and migrations</a> to understand which solution best fits your use case.</p> 
<h2>Solution overview</h2> 
<p>MSK Replicator supports Kafka deployments running version 2.8.1 or later as a source, including 3rd party managed Kafka services, self-managed Kafka, and on-premises or third-party cloud-hosted Kafka. MSK Replicator automatically handles data transfer, uses SASL/SCRAM authentication with SSL encryption, and maintains consumer group positions across both clusters. If you do not use SASL/SCRAM today, this can be configured as a new listener used for MSK Replicator allowing current clients to use their existing authentication mechanisms alongside MSK Replicator.</p> 
<h2>Prerequisites</h2> 
<p>To follow along with this walkthrough, you need the following resources in place:</p> 
<ul> 
 <li>A source Kafka cluster using <a href="https://kafka.apache.org/community/downloads/#281" rel="noopener noreferrer" target="_blank">Kafka version 2.8.1</a> or above</li> 
 <li>Network connectivity between your external Kafka cluster and AWS (for example, using <a href="https://aws.amazon.com/directconnect/" rel="noopener noreferrer" target="_blank">AWS Direct Connect</a>, <a href="https://aws.amazon.com/vpn/" rel="noopener noreferrer" target="_blank">Site-to-Site VPN</a>, or <a href="https://aws.amazon.com/vpc/" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud</a> (VPC) <a href="https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html">peering</a> or <a href="https://aws.amazon.com/transit-gateway/">AWS Transit Gateway</a> for connections between AWS VPCs) so that MSK Replicator can reach your source brokers</li> 
 <li>SASL/SCRAM authentication configured on your external cluster (SHA-256 or SHA-512), which MSK Replicator uses to authenticate with external clusters</li> 
 <li>An admin user configured on your external cluster with permissions to describe the external cluster and create and modify users/ACLs</li> 
 <li>An Amazon MSK Express cluster with <a href="https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html" rel="noopener noreferrer" target="_blank">IAM authentication enabled</a> to serve as your target</li> 
 <li><a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> configured to store your SASL/SCRAM credentials for the external cluster so that MSK Replicator can securely retrieve them at runtime</li> 
 <li>An <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> log group for MSK Replicator logs</li> 
 <li>Appropriate <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator-create-iam-perms.html" rel="noopener noreferrer" target="_blank">IAM permissions for creating and managing MSK Replicator</a> resources</li> 
</ul> 
<h2>Setting up replication</h2> 
<h3>Step 1: Configure network connectivity</h3> 
<p>You can set up network connectivity between your external Kafka cluster and your AWS VPC using methods such as AWS Direct Connect for dedicated network connections, AWS Site-to-Site VPN for encrypted connections over the internet, and AWS VPC peering or AWS Transit Gateway for connections between AWS VPCs. Verify that IP routing and DNS resolution are properly configured between your external cluster and AWS.</p> 
<p>To verify IP routing and DNS resolution, connect to your external Kafka cluster from inside of your VPC by using the Kafka CLI to list topics on the external cluster. If you can list topics from your VPC using the Kafka CLI, this means DNS resolution and IP routing are working successfully. If it fails, work with your network admins to troubleshoot network connectivity issues.</p> 
<h3>Step 2: Configure external cluster</h3> 
<p>In this step, you will set up authentication on your external Kafka cluster and store the credentials in AWS Secrets Manager so that MSK Replicator can connect securely.</p> 
<h4>Configure authentication</h4> 
<p>Using the external cluster admin user, configure SASL/SCRAM authentication for MSK Replicator using SHA-256 or 512 on your external Kafka cluster. Create a SASL/SCRAM user for MSK Replicator and give the user the following ACL permissions:</p> 
<ul> 
 <li><strong>Topic operations –</strong> Alter, AlterConfigs, Create, Describe, DescribeConfigs, Read, Write</li> 
 <li><strong>Group operations –</strong> Read, Describe</li> 
 <li><strong>Cluster operations –</strong> Create, ClusterAction, Describe, DescribeConfigs</li> 
</ul> 
<h4>Configure SecretsManager</h4> 
<p>AWS Secrets Manager stores your SASL/SCRAM credentials securely so that MSK Replicator can retrieve them at runtime. The secret must use JSON format and have the following keys:</p> 
<ul> 
 <li><code><strong>username</strong></code> – The SCRAM username that you configured in the authentication step above</li> 
 <li><code><strong>password</strong></code> – The SCRAM password that you configured in the authentication step above</li> 
 <li><code><strong>certificate</strong></code> – The public root CA certificate (the top-level certificate authority that issued your cluster’s TLS certificate) and the intermediate CA chain (intermediate certificates between the root and your cluster’s certificate), used for SSL handshakes with the external cluster</li> 
</ul> 
<p>Optionally, you may create separate secrets for SCRAM credentials and the SSL certificate. This approach is useful when secrets for SCRAM credentials and certificates are provisioned in different stages, such as in Infrastructure as Code (IaC) pipelines.</p> 
<h4>Retrieve the cluster ID</h4> 
<p>As the admin user, use the <a href="https://downloads.apache.org/kafka/" rel="noopener noreferrer" target="_blank">Kafka CLI tools</a> to retrieve the cluster ID of your external cluster. Run the following command, replacing <code>your-broker-host:9096</code> with the address of one of your external cluster’s bootstrap servers:</p> 
<pre><code class="lang-code">bin/kafka-cluster.sh cluster-id --bootstrap-server your-broker-host:9096 --config admin.properties</code></pre> 
<p>The command returns a cluster ID string such as <code>lkc-abc123</code>. Take note of this value because you will need it when creating the replicator in Step 4.</p> 
<h3>Step 3: Create your MSK Express target cluster</h3> 
<p>With your external cluster configured, you can now set up the target. Create an Amazon MSK Express cluster with IAM authentication enabled. Make sure that the cluster is in subnets that have access to <a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> endpoints. See <a href="https://docs.aws.amazon.com/msk/latest/developerguide/getting-started.html" rel="noopener noreferrer" target="_blank">Get started using Amazon MSK</a> for more information on creating an MSK cluster.</p> 
<h3>Step 4: Create the replicator</h3> 
<p>Now that both clusters are ready, you can connect them by setting up the MSK Replicator with the appropriate IAM role and replication configuration.</p> 
<h4>Set up an IAM role for MSK Replicator</h4> 
<p>MSK Replicator needs an IAM role to interact with your MSK Express cluster and retrieve secrets. Set up a service execution IAM role with a trust policy allowing <code>kafka.amazonaws.com</code> and attach the <code>AWSMSKReplicatorExecutionRole</code> permissions policy. Take note of the role ARN for creating the replicator.</p> 
<p>Create and attach a policy for accessing your Secrets Manager secrets and reading/writing data in your MSK cluster. See <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions_create-policies.html" rel="noopener noreferrer" target="_blank">Creating roles and attaching policies (console)</a> for more information on creating IAM roles and policies.</p> 
<p>The following is an example policy for reading and writing data to your MSK cluster and reading KMS-encrypted Secrets Manager secrets:</p> 
<pre><code class="lang-json">{&nbsp;
&nbsp;&nbsp;&nbsp; "Version": "2012-10-17",&nbsp;
&nbsp;&nbsp;&nbsp; "Statement": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Sid": "SecretsManagerAccess",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Effect": "Allow",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Action": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "secretsmanager:GetSecretValue",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "secretsmanager:DescribeSecret"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Resource": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "&lt;SCRAM_SECRET_ARN&gt;",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "&lt;CERT_SECRET_ARN&gt;"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ]&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Sid": "KMSDecrypt",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Effect": "Allow",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Action": "kms:Decrypt",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Resource": "&lt;SECRETSMANAGER_KMS_KEY_ARN&gt;"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Sid": "TargetClusterAccess",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Effect": "Allow",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Action": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:Connect",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:DescribeCluster",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:AlterCluster",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:DescribeClusterDynamicConfiguration",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:AlterClusterDynamicConfiguration",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:DescribeTopic",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:CreateTopic",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:AlterTopic",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:DescribeTopicDynamicConfiguration",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:AlterTopicDynamicConfiguration",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:WriteData",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:WriteDataIdempotently",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:ReadData",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:DescribeGroup",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "kafka-cluster:AlterGroup"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Resource": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:cluster/&lt;MSK_CLUSTER_NAME&gt;*/*",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:topic/&lt;MSK_CLUSTER_NAME&gt;/*",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:group/&lt;MSK_CLUSTER_NAME&gt;*/*"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ]&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Sid": "CloudWatchLogsAccess",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Effect": "Allow",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Action": [&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "logs:CreateLogStream",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "logs:PutLogEvents",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "logs:DescribeLogStreams"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Resource": "&lt;MSK_REPLICATOR_LOG_GROUP_ARN&gt;"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }&nbsp;
&nbsp;&nbsp;&nbsp; ]&nbsp;
}
</code></pre> 
<h4>Create the replicator for external to MSK replication</h4> 
<p>Use the AWS CLI, API, or Console to create your replicator. Here’s an example using the AWS CLI:</p> 
<pre><code class="lang-bash">aws kafka create-replicator \
&nbsp; --replicator-name external-to-msk \
&nbsp; --service-execution-role-arn "arn:aws:iam::123456789012:role/MSKReplicatorRole" \
&nbsp; --kafka-clusters file://./kafka-clusters.json \
&nbsp; --replication-info-list file://./replication-info.json \
&nbsp; --log-delivery file://./log-delivery.json \
&nbsp; --region us-east-1</code></pre> 
<p>The <code>kafka-clusters.json</code> file defines the source and target Kafka cluster connection information, <code>replication-info.json</code> specifies which topics to replicate and how to handle consumer group offset synchronization, and <code>log-delivery.json</code> specifies the CloudWatch logging configuration. The following tables describe the required parameters:</p> 
<p><strong><em>CLI inputs:</em></strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CLI Parameter</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Description</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Example</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">replicator-name</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The name of the replicator</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">external-to-msk</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">service-execution-role-arn</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The ARN for the service execution IAM role you created</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">arn:aws:iam::123456789012:role/MSKReplicatorRole</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">kafka-clusters</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The Kafka cluster connection info</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">See below</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">replication-info-list</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The replication configuration</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">See below</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">log-delivery</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The logging configuration</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">See below</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong><em>Key <code>kafka-clusters.json</code> inputs:</em></strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CLI Parameter</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Description</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Example</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ApacheKafkaClusterId</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The cluster ID retrieved in Step 2</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">lkc-abc123</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">RootCaCertificate</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The Secrets Manager ARN containing the public CA certificate and intermediate CA chain</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">arn:aws:secretsmanager:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:secret:my-cert</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">MskClusterArn</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The ARN for the MSK Express cluster</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:cluster/my-cluster/abc-123</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">SecretArn</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The Secrets Manager ARN containing the SASL/SCRAM username and password</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">arn:aws:secretsmanager:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:secret:my-creds</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">SecurityGroupIds</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The security group IDs for MSK Replicator</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">sg-0123456789abcdef0</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong><em>Key <code>replication-info.json</code> inputs:</em></strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CLI Parameter</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Description</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Example</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">TargetCompressionType</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The compression type to use for replicating data</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">LZ4</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">TopicsToReplicate</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The list of topics to replicate (use [“.*”] for all topics)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">[“my-topic”]</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ConsumerGroupsToReplicate</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The list of consumer groups to replicate</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">[“my-group”]</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">StartingPosition</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The point in the Kafka topics to begin replication from (either EARLIEST or LATEST)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">EARLIEST</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ConsumerGroupOffsetSyncMode</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Whether or not to use enhanced bidirectional consumer group offset synchronization</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ENHANCED</td> 
  </tr> 
 </tbody> 
</table> 
<p>Note that <code>startingPosition</code> is set to <code>EARLIEST</code> in the configuration below, which means the replicator begins reading from the oldest available offset on each topic. This is the recommended setting for migrations to avoid data loss.</p> 
<p><strong><em>Key <code>log-delivery.json</code> inputs:</em></strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CLI Parameter</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Description</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Example</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Enabled</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Allows you to enable CloudWatch logging</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">true</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">LogGroup</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">The CloudWatch logs log group name to log to</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">/msk/replicator/my-replicator</td> 
  </tr> 
 </tbody> 
</table> 
<p>Additional log delivery methods for <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a> and <a href="https://aws.amazon.com/firehose/" rel="noopener noreferrer" target="_blank">Amazon Data Firehose</a> are supported. In this post, we use CloudWatch logging.</p> 
<p>The configs should look like the following for external to MSK replication.</p> 
<p><strong><code>kafka-clusters.json:</code></strong></p> 
<pre><code class="lang-json">[&nbsp;
&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp; "ApacheKafkaCluster": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "ApacheKafkaClusterId": "lkc-abc123",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "BootstrapBrokerString": "broker1.example.com:9096"&nbsp;
&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp; "ClientAuthentication": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "SaslScram": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Mechanism": "SHA512",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "SecretArn": "arn:aws:secretsmanager:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:secret:my-creds"&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }&nbsp;
&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp; "EncryptionInTransit": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "EncryptionType": "TLS",&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "RootCaCertificate": "arn:aws:secretsmanager:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:secret:my-cert"&nbsp;
&nbsp;&nbsp;&nbsp; }&nbsp;
&nbsp; },&nbsp;
&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp; "AmazonMskCluster": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "MskClusterArn": "arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:cluster/my-cluster/abc-123"&nbsp;
&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp; "VpcConfig": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "SecurityGroupIds": ["sg-0123456789abcdef0"],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "SubnetIds": ["subnet-abc123", "subnet-abc124", "subnet-abc125"]&nbsp;
&nbsp;&nbsp;&nbsp; }&nbsp;
&nbsp; }&nbsp;
]&nbsp;</code></pre> 
<p><strong><code>replication-info.json:&nbsp;</code></strong></p> 
<pre><code class="lang-json">[&nbsp;
&nbsp; {&nbsp;
&nbsp;&nbsp;&nbsp; "SourceKafkaClusterId": "lkc-abc123",&nbsp;
&nbsp;&nbsp;&nbsp; "TargetKafkaClusterArn": "arn:aws:kafka:&lt;REGION&gt;:&lt;ACCOUNT_ID&gt;:cluster/my-cluster/abc-123",&nbsp;
&nbsp;&nbsp;&nbsp; "TargetCompressionType": "LZ4",&nbsp;
&nbsp;&nbsp;&nbsp; "TopicReplication": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "TopicsToReplicate": ["my-topic"],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "CopyTopicConfigurations": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "CopyAccessControlListsForTopics": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "DetectAndCopyNewTopics": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "StartingPosition": {"Type": "EARLIEST"},&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "TopicNameConfiguration": {"Type": "IDENTICAL"}&nbsp;
&nbsp;&nbsp;&nbsp; },&nbsp;
&nbsp;&nbsp;&nbsp; "ConsumerGroupReplication": {&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "ConsumerGroupsToReplicate": ["my-group"],&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "SynchroniseConsumerGroupOffsets": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "DetectAndCopyNewConsumerGroups": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "ConsumerGroupOffsetSyncMode": "ENHANCED"&nbsp;
&nbsp;&nbsp;&nbsp; }&nbsp;
&nbsp; }&nbsp;
]&nbsp;</code></pre> 
<p><strong><code>log-delivery.json:&nbsp;</code></strong></p> 
<pre><code class="lang-json">{&nbsp;
&nbsp; "ReplicatorLogDelivery": {
&nbsp;&nbsp;&nbsp;&nbsp; "CloudWatchLogs": {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "Enabled": true,&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; "LogGroup": "&lt;LOG_GROUP_NAME&gt;"
&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp; }&nbsp;
}</code></pre> 
<h4>Configure bidirectional replication from MSK to the external cluster</h4> 
<p>To enable bidirectional replication, create a second replicator that replicates in the opposite direction. Use the same IAM role and network configuration from Step 4, but swap the source and target. Replace <code>SourceKafkaClusterId</code> with <code>TargetKafkaClusterId</code> and <code>TargetKafkaClusterArn</code> with <code>SourceKafkaClusterArn</code> in a new <code>msk-to-external-replication-info.json</code> file:</p> 
<pre><code class="lang-bash">aws kafka create-replicator \
  --replicator-name msk-to-external \
  --service-execution-role-arn "arn:aws:iam::123456789012:role/MSKReplicatorRole" \
  --kafka-clusters file:///./kafka-clusters.json \
  --replication-info-list file:///./msk-to-external-replication-info.json \
  --log-delivery file:///./log-delivery.json \
  --region us-east-1</code></pre> 
<h2>Monitoring replication health</h2> 
<p>Monitor your replication using Amazon CloudWatch metrics. Three key metrics to understand are <code>MessageLag</code>, <code>SumOffsetLag</code>, and <code>ReplicationLatency</code>. <code>MessageLag</code> measures how far behind the replicator is from the external cluster in terms of messages not yet replicated, while <code>SumOffsetLag</code> measures how far behind a consumer group is from the latest message in a topic. <code>ReplicationLatency</code> is the amount of latency between the source and target clusters in data replication. When the three reach a sustained low level, your clusters are fully synchronized for both data and consumer group offsets.</p> 
<p>To troubleshoot MSK Replicator replication or errors, use the CloudWatch logs to get more details about the health of the replicator. MSK Replicator logs status and troubleshooting information which can be helpful in diagnosing issues like connectivity, authentication, and SSL errors.</p> 
<p>Note that the replication is asynchronous, so there will be some lag during replication. The lag will reach zero once a client is shut down during migration to the target cluster. This takes about 30 seconds under normal operations, allowing a low downtime migration without data loss. If your lag is continually increasing or does not reach a sustained low level, this indicates that you have insufficient partitions for high-throughput replication. Refer to <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator-troubleshooting.html">Troubleshoot MSK Replicator</a> for more information on troubleshooting replication throughput and lag.</p> 
<p>Key metrics include:</p> 
<ul> 
 <li><strong>MessageLag –</strong> Monitors the sync between the MSK Replicator and the source cluster. MessageLag indicates the lag between the messages produced to the source cluster and messages consumed by the replicator. It is not the lag between the source and target cluster.</li> 
 <li><strong>ReplicationLatency –</strong> Time taken for records to replicate from source to target cluster (ms)</li> 
 <li><strong>ReplicatorThroughput –</strong> Average number of bytes replicated per second</li> 
 <li><strong>ReplicatorFailure –</strong> Number of failures the replicator is experiencing</li> 
 <li><strong>KafkaClusterPingSuccessCount –</strong> Connection health indicator (1 = healthy, 0 = unhealthy)</li> 
 <li><strong>ConsumerGroupCount –</strong> Total consumer groups being synchronized</li> 
 <li><strong>ConsumerGroupOffsetSyncFailure –</strong> Failures during offset synchronization</li> 
 <li><strong>AuthError –</strong> Number of connections with failed authentication per second, by cluster</li> 
 <li><strong>ThrottleTime –</strong> Average time in ms a request was throttled by brokers, by cluster</li> 
 <li><strong>SumOffsetLag –</strong> Aggregated offset lag across partitions for a consumer group on a topic (MSK cluster-level metric)</li> 
</ul> 
<p>For more details on these metrics, see the <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator-monitor.html" rel="noopener noreferrer" target="_blank">MSK Replicator metrics documentation</a>.</p> 
<p>Your applications are ready to migrate when the following conditions are met. For most workloads, you should expect these metrics to stabilize within a few hours of starting replication. High-throughput clusters may take longer depending on topic volume and partition count.</p> 
<ul> 
 <li><strong>ReplicatorFailure</strong> = 0</li> 
 <li><strong>ConsumerGroupOffsetSyncFailure</strong> = 0</li> 
 <li><strong>KafkaClusterPingSuccessCount</strong> = 1 for both source and target clusters</li> 
 <li><strong>MessageLag</strong> &lt; 1,000 
  <ul> 
   <li>Your sustained lag may be lower or higher depending on your throughput per partition, message size, and other factors</li> 
   <li>Sustained high message lag usually indicates insufficient partitions for high-throughput replication</li> 
  </ul> </li> 
 <li><strong>ReplicationLatency</strong> &lt; 90 seconds 
  <ul> 
   <li>Your sustained latency may be lower or higher depending on your throughput per partition, message size, and other factors</li> 
   <li>Sustained high latency usually indicates insufficient partitions for high-throughput replication</li> 
  </ul> </li> 
 <li><strong>SumOffsetLag</strong> is at a sustained low level on both clusters 
  <ul> 
   <li>Offset values on the two clusters may not be numerically identical.</li> 
   <li>MSK Replicator translates offsets between clusters so that consumers resume from the correct position, but the raw offset numbers can differ due to how offset translation works. What matters is that SumOffsetLag is at a sustained low level.</li> 
  </ul> </li> 
 <li><strong>ConsumerGroupCount</strong> (MSK) = Expected count (external cluster) 
  <ul> 
   <li>If ConsumerGroupCount is zero or does not match the expected count, then there is an issue in the Replicator configuration or a permissions issue preventing consumer group synchronization</li> 
  </ul> </li> 
</ul> 
<h2>Migrating your applications</h2> 
<p>With bidirectional consumer offset synchronization, you can migrate your producers and consumers regardless of order. Start by monitoring replication metrics until they reach the target values described in the previous section. Then migrate your applications (producers or consumers) to use the MSK Express cluster endpoints and verify that they are producing and consuming as expected. If you encounter issues, you can roll back by switching applications back to the external cluster. The consumer offset synchronization makes sure that your applications resume from their last committed position regardless of which cluster they connect to.</p> 
<p>For a comprehensive, hands-on walkthrough of the end-to-end migration process, explore the <a href="https://catalog.workshops.aws/msk-migration-lab" rel="noopener noreferrer" target="_blank">MSK Migration Workshop</a>, which provides step-by-step guidance for migrating your Kafka workloads to Amazon MSK.</p> 
<h2>Security considerations</h2> 
<p>MSK Replicator uses SASL/SCRAM authentication with SSL encryption for secure data transfer between your external cluster and AWS. The solution supports both publicly trusted certificates and private or self-signed certificates. Credentials are stored securely in <a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a>, and the target MSK Express cluster uses <a href="https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html" rel="noopener noreferrer" target="_blank">IAM authentication</a> for access control.</p> 
<p>When configuring security, keep the following in mind:</p> 
<ul> 
 <li>Make sure that the IAM role you create in Step 4 follows the principle of least privileges. Only attach <code>AWSMSKReplicatorExecutionRole</code> and an IAM policy for Secrets Manager with least-privileges access to read secret values and avoid adding broader permissions.</li> 
 <li>Verify that your Secrets Manager secret is encrypted with an AWS KMS key that the MSK Replicator service execution role has permission to decrypt.</li> 
 <li>Confirm that the security groups assigned to MSK Replicator allow outbound traffic to your external cluster’s broker ports (typically 9096 for SASL/SCRAM with TLS) and to the MSK Express cluster.</li> 
 <li>Rotate your SASL/SCRAM credentials periodically and update the corresponding Secrets Manager secret. MSK Replicator picks up the new credentials automatically on the next connection attempt.</li> 
</ul> 
<p>Under the <a href="https://aws.amazon.com/compliance/shared-responsibility-model/" rel="noopener noreferrer" target="_blank">AWS shared responsibility model</a>, AWS is responsible for securing the underlying infrastructure that runs MSK Replicator, including the compute, storage, and networking resources. You are responsible for configuring authentication mechanisms (SASL/SCRAM), managing credentials in AWS Secrets Manager, configuring network security (security groups and VPC settings), implementing IAM policies following least privilege, and rotating credentials. For more information, see <a href="https://docs.aws.amazon.com/msk/latest/developerguide/security.html" rel="noopener noreferrer" target="_blank">Security in Amazon MSK</a> in the Amazon MSK Developer Guide.</p> 
<h2>Cleanup</h2> 
<p>To avoid ongoing charges, delete the resources you created during this walkthrough. Start by deleting the replicators first, because they depend on the other resources:</p> 
<p><code>aws kafka delete-replicator --replicator-arn &lt;replicator-arn&gt;</code></p> 
<p>After both replicators are deleted, you can remove the following resources if they were created solely for this walkthrough:</p> 
<ol> 
 <li>The MSK Express cluster (deleting a cluster also removes its stored data, so verify that your applications have fully migrated before proceeding)</li> 
 <li>The Secrets Manager secrets containing your SASL/SCRAM credentials and certificates</li> 
 <li>The IAM role and policies created for MSK Replicator</li> 
</ol> 
<p>You can verify that a replicator has been fully deleted by running <code>aws kafka list-replicators</code> and confirming it no longer appears in the output.</p> 
<h2>Conclusion</h2> 
<p>Amazon MSK Replicator simplifies the process of migrating to Amazon MSK Express brokers and establishes hybrid Kafka architectures. The fully managed service alleviates the operational complexity of managing replication while bidirectional consumer offset synchronization enables flexible, low-risk application migration.</p> 
<h3>Next Steps</h3> 
<p>To get started using MSK Replicator to migrate applications to MSK Express brokers, use the <a href="https://catalog.workshops.aws/msk-migration-lab" rel="noopener noreferrer" target="_blank">MSK Migration Workshop</a> for a hands-on, end-to-end migration walkthrough. The <a href="https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator.html" rel="noopener noreferrer" target="_blank">Amazon MSK Replicator documentation</a> includes detailed configuration details to help configure MSK Replicator for your use case. From there, use MSK Replicator to migrate your Apache Kafka workloads to MSK Express broker.</p> 
<p>Once your migration is complete, consider exploring multi-region replication patterns for disaster recovery, or integrating your MSK Express cluster with AWS analytics services such as <a href="https://aws.amazon.com/firehose/" rel="noopener noreferrer" target="_blank">Amazon Data Firehose</a> and <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a>. If you need help planning your migration, reach out to your AWS account team, <a href="https://aws.amazon.com/support/" rel="noopener noreferrer" target="_blank">AWS Support</a> or <a href="https://aws.amazon.com/professional-services/" rel="noopener noreferrer" target="_blank">AWS Professional Services</a>.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone wp-image-90062 size-thumbnail" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/10/ankitams-100x133.jpg" width="100" />
  </div> 
  <h3 class="lb-h4">Ankita Mishra</h3> 
  <p><a href="https://www.linkedin.com/in/ankitamishra05" rel="noopener" target="_blank">Ankita</a> is a Product Manager for Amazon Managed Streaming for Apache Kafka. She works closely with AWS customers to understand their needs for real-time analytics and high throughput, low latency streaming workloads. Working backwards from their needs, she helps drive the MSK roadmap and deliver new innovations that help AWS customers focus on building novel streaming applications.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-89475" height="107" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/24/bdb-5775-mmehrten-headshot.png" width="100" />
  </div> 
  <h3 class="lb-h4">Mazrim Mehrtens</h3> 
  <p><a href="https://www.linkedin.com/in/mmehrtens/" rel="noopener" target="_blank">Mazrim</a> is a Sr. Specialist Solutions Architect for messaging and streaming workloads. Mazrim works with customers to build and support systems that process and analyze terabytes of streaming data in real time, run enterprise Machine Learning pipelines, and create systems to share data across teams seamlessly with varying data toolsets and software stacks.</p> 
 </div> 
</footer>
