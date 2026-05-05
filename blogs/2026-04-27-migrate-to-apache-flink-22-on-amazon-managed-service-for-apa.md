---
title: "Migrate to Apache Flink 2.2 on Amazon Managed Service for Apache Flink"
url: "https://aws.amazon.com/blogs/big-data/migrate-to-apache-flink-2-2-on-amazon-managed-service-for-apache-flink/"
date: "Mon, 27 Apr 2026 17:57:34 +0000"
author: "Francisco Morillo"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Migrating to <a href="https://aws.amazon.com/managed-service-apache-flink/" rel="noopener noreferrer" target="_blank">Apache Flink 2.2</a> on&nbsp;<a href="https://aws.amazon.com/managed-service-apache-flink/" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink&nbsp;</a>gives you access to Java 17 runtime, faster checkpoints and recovery through RocksDB 8.10.0, and SQL-native artificial intelligence and machine learning (AI/ML) inference. If you run Flink 1.x today, you might be dealing with an aging Java 11 runtime that will no longer receive standard support by the end of this year, slower state backend performance, and a fragmented API surface split across DataSet, DataStream, and legacy connector interfaces. Flink 2.2 addresses these gaps in a single major version upgrade.</p> 
<p><a href="https://flink.apache.org/" rel="noopener noreferrer" target="_blank">Apache Flink&nbsp;</a>is an open source distributed processing engine for stream and batch data, with first-class support for stateful processing and event-time semantics. Amazon Managed Service for Apache Flink removes the operational overhead of running Flink. You provide your application code, and the service provisions, scales, checkpoints, and patches the infrastructure for you.</p> 
<p>In this post, we explain what’s new in <a href="https://docs.aws.amazon.com/managed-flink/latest/java/flink-2-2.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink 2.2</a>, provide a guided migration using CLI commands, console instructions, and code examples, and show you how to monitor the upgrade and roll back if needed.</p> 
<p><strong>Before you upgrade:</strong>&nbsp;Flink 2.2 removes the DataSet API, drops Java 11 support, and replaces legacy connector interfaces. We recommend reviewing the&nbsp;<a href="https://docs.aws.amazon.com/managed-flink/latest/java/flink-2-2-upgrade-guide.html" rel="noopener noreferrer" target="_blank">Upgrading to Flink 2.2: Complete Guide&nbsp;</a>and the&nbsp;<a href="https://docs.aws.amazon.com/managed-flink/latest/java/state-compatibility.html" rel="noopener noreferrer" target="_blank">State Compatibility Guide for Flink 2.2 Upgrade</a>s&nbsp;before upgrading production applications.</p> 
<h2>What’s new in Amazon Managed Service for Apache Flink 2.2</h2> 
<p>This release spans runtime upgrades, SQL, and Table API capabilities. The following sections break down each area.</p> 
<h3>Runtime and performance</h3> 
<p>These changes improve application performance and bring your runtime up to current standards.</p> 
<ul> 
 <li><strong><a href="https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/java_compatibility/" rel="noopener noreferrer" target="_blank">Java 17 runtime</a> –</strong>&nbsp;Flink 2.2 requires Java 17. Build your application code with JDK 17 for better garbage collection, a more secure runtime, and modern language features like sealed classes and records. Java 11 is no longer supported.</li> 
 <li><strong><a href="https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/dev/python/overview/" rel="noopener noreferrer" target="_blank">Python 3.12</a> –</strong>&nbsp;Flink 2.2 requires Python 3.9+, with Python 3.12 as the default. Python 3.8 is no longer supported.</li> 
 <li><strong>RocksDB 8.10.0 –</strong>&nbsp;Your stateful applications benefit from improved I/O performance with the upgraded state backend, resulting in faster checkpoints and recovery.</li> 
 <li><strong>Dedicated collection serializers –</strong>&nbsp;Improved serializers for Map, List, and Set types reduce serialization overhead, which lowers checkpoint sizes for applications that use these data structures frequently.</li> 
 <li><strong>Kryo 5.6 –</strong>&nbsp;Kryo upgrades from version 2.24–5.6. This has state compatibility implications covered in the migration section.</li> 
</ul> 
<h3>SQL and Table API highlights</h3> 
<p>With Flink 2.2, you can:</p> 
<ul> 
 <li>Call Machine Leaning (ML) models directly from SQL using <a href="https://nightlies.apache.org/flink/flink-docs-release-2.2/" rel="noopener noreferrer" target="_blank">ML_PREDICT</a> and CREATE MODEL</li> 
 <li>Work with semistructured data through the native <a href="https://nightlies.apache.org/flink/flink-docs-master/docs/sql/reference/data-types/" rel="noopener noreferrer" target="_blank">VARIANT type</a></li> 
 <li>Build stateful event-driven logic in SQL with <a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/table/functions/ptfs/" rel="noopener noreferrer" target="_blank">ProcessTableFunction</a></li> 
 <li>Run more efficient streaming joins with <a href="https://nightlies.apache.org/flink/flink-docs-stable/api/java/org/apache/flink/table/runtime/operators/join/stream/StreamingMultiJoinOperator.html" rel="noopener noreferrer" target="_blank">StreamingMultiJoinOperator</a> and <a href="https://flink.apache.org/2025/12/04/apache-flink-2.2.0-advancing-real-time-data--ai-and-empowering-stream-processing-for-the-ai-era/#delta-join" rel="noopener noreferrer" target="_blank">Delta Join</a></li> 
</ul> 
<p>For details on these features, see the&nbsp;<a href="https://flink.apache.org/2025/12/04/apache-flink-2.2.0-advancing-real-time-data--ai-and-empowering-stream-processing-for-the-ai-era/" rel="noopener noreferrer" target="_blank">Apache Flink 2.2 release documentation</a>.</p> 
<h2>Migrating from Flink 1.x to 2.2</h2> 
<h3>In-place version upgrades</h3> 
<p>You can upgrade a running Flink 1.x application to 2.2 using the <a href="https://docs.aws.amazon.com/managed-flink/latest/java/how-in-place-version-upgrades.html" rel="noopener noreferrer" target="_blank">UpdateApplication API</a>, the AWS Management Console, AWS CloudFormation, the AWS SDK, and Terraform Modules. The upgrade preserves your application configuration, logs, metrics, tags, and, if your state and binaries are compatible.</p> 
<h3>Auto-rollback</h3> 
<p>With <a href="https://docs.aws.amazon.com/managed-flink/latest/java/troubleshooting-system-rollback.html" rel="noopener noreferrer" target="_blank">auto-rollback</a> turned on, binary incompatibilities detected during job startup trigger an automatic revert to the previous Flink version within minutes, with no manual intervention required. For state incompatibilities that surface as restart loops after a successful upgrade, invoke the Rollback API to return to your previous version and state.</p> 
<h3>Unsupported open source features</h3> 
<p>The following Flink 2.2 features aren’t currently supported in Amazon Managed Service for Apache Flink because they’re still considered experimental: Materialized Tables, ForSt State Backend (disaggregated state storage), Java 21, and custom metric reporters/telemetry configurations. We continue to evaluate these features as they mature in the Apache Flink project and will share updates on availability. You can have a closer look to which features are supported in <a href="https://docs.aws.amazon.com/managed-flink/latest/java/flink-2-2.html#flink-2-2-supported-features" rel="noopener noreferrer" target="_blank">Apache Flink 2.2 features supported</a></p> 
<p>Now that you know what’s changed, the next section walks through the migration process.</p> 
<h2>Prerequisites</h2> 
<p>Before starting the migration, confirm that you have the following in place:</p> 
<ul> 
 <li>An existing Apache Flink 1.x application running on Amazon Managed Service for Apache Flink.</li> 
 <li>JDK 17 installed in your local build environment.</li> 
 <li>The AWS Command Line Interface (AWS CLI) installed and configured with permissions to call the&nbsp;kinesisanalyticsv2&nbsp;APIs (UpdateApplication, CreateApplicationSnapshot, DescribeApplication, RollbackApplication).</li> 
 <li>An Amazon Simple Storage Service (Amazon S3) bucket to upload your updated application JAR.</li> 
</ul> 
<p>We recommend testing each phase on a non-production replica of your application before applying the same steps to production.</p> 
<h3>Step 1: Update your application code</h3> 
<p>Start by updating your Flink dependencies to version 2.2.0 and replacing deprecated APIs. The following sections show the most common changes.</p> 
<p><strong>Update your pom.xml:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-html">&lt;properties&gt;
    &lt;flink.version&gt;2.2.0&lt;/flink.version&gt;
    &lt;java.version&gt;17&lt;/java.version&gt;
&lt;/properties&gt;</code></pre> 
</div> 
<p><strong>Replace legacy Kinesis connectors:</strong></p> 
<p>Flink 2.2 removes the&nbsp;FlinkKinesisConsumer&nbsp;and&nbsp;FlinkKinesisProducer&nbsp;classes. The following example shows how to migrate to the FLIP-27 based&nbsp;KinesisStreamsSource.Before (Flink 1.x):</p> 
<div class="hide-language"> 
 <pre><code class="lang-java">FlinkKinesisConsumer&lt;String&gt; consumer = new FlinkKinesisConsumer&lt;&gt;(
    "my-stream",
    new SimpleStringSchema(),
    consumerConfig);
env.addSource(consumer);</code></pre> 
</div> 
<p>After (Flink 2.2):</p> 
<div class="hide-language"> 
 <pre><code class="lang-java">KinesisStreamsSource&lt;String&gt; source = KinesisStreamsSource.&lt;String&gt;builder()
    .setStreamArn("arn:aws:kinesis:us-east-1:123456789012:stream/my-stream")
    .setDeserializationSchema(new SimpleStringSchema())
    .build();
env.fromSource(source, WatermarkStrategy.noWatermarks(), "Kinesis Source");</code></pre> 
</div> 
<p><strong>Update connector dependencies:</strong></p> 
<p>The following AWS connectors have Flink 2.x-compatible releases:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <thead> 
  <tr> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Connector</strong></th> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Flink 2.x Artifact</strong></th> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Version</strong></th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Apache Kafka</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">flink-connector-kafka</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4.0.0-2.0</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon Kinesis Data Streams</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">flink-connector-aws-kinesis-streams</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6.0.0-2.0</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon Data Firehose</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">flink-connector-aws-kinesis-firehose</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6.0.0-2.0</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon DynamoDB</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">flink-connector-dynamodb</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6.0.0-2.0</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon Simple Queue Service (Amazon SQS)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">flink-connector-sqs</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">6.0.0-2.0</td> 
  </tr> 
 </tbody> 
</table> 
<p>During writing, the JDBC, OpenSearch, and Prometheus connectors don’t yet have Flink 2.x-compatible releases. For the latest versions, see the&nbsp;<a href="https://docs.aws.amazon.com/managed-flink/latest/java/how-flink-connectors.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink connector documentation</a>.</p> 
<p>Beyond connector updates, make the following code changes:</p> 
<ul> 
 <li>Replace DataSet API usage with the DataStream API or Table API/SQL.</li> 
 <li>Replace Scala API usage with the Java API.</li> 
 <li>Verify that your build targets JDK 17.</li> 
</ul> 
<p>Build your updated application JAR and upload it to Amazon S3 with a different file name than your current JAR (for example,&nbsp;my-app-flink-2.2.jar).</p> 
<h3>Step 2: Check state compatibility</h3> 
<p>Before upgrading, assess whether your application state is compatible with Flink 2.2. The Kryo upgrade from version 2.24 to 5.6 changes the binary format of serialized state. Applications using POJOs with Java collections (HashMap, ArrayList, HashSet) are the most common source of incompatibility.</p> 
<p><strong>Quick compatibility check:</strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <thead> 
  <tr> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Serialization type</strong></th> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Compatible?</strong></th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Avro (SpecificRecord, GenericRecord)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="✅" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2705.png" style="height: 1em;" /> Yes</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Protobuf</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="✅" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2705.png" style="height: 1em;" /> Yes</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">POJOs without collections</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="✅" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2705.png" style="height: 1em;" /> Yes</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Custom TypeSerializers (no Kryo delegation)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="✅" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/2705.png" style="height: 1em;" /> Yes</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">POJOs with Java collections</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="❌" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/274c.png" style="height: 1em;" /> No</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Scala case classes</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="❌" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/274c.png" style="height: 1em;" /> No</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Types using Kryo fallback</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><img alt="❌" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/274c.png" style="height: 1em;" /> No</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Check your logs for Kryo fallback:</strong></p> 
<p>Search your application logs for this pattern, which indicates a type is falling back to Kryo serialization:<code>Class class &lt;className&gt; cannot be used as a POJO type</code></p> 
<h3>Step 3: Turn on auto-rollback and automatic snapshots</h3> 
<p>Turn on auto-rollback so the service automatically reverts to the previous version if the upgrade fails. Also, verify that automatic snapshots are turned on. The service takes a snapshot before the upgrade that serves as your rollback point.</p> 
<p><strong>Check current settings:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-css">aws kinesisanalyticsv2 describe-application \
    --application-name MyApplication \
    --query 'ApplicationDetail.ApplicationConfigurationDescription.{
        AutoRollback: ApplicationSystemRollbackConfigurationDescription.RollbackEnabled,
        AutoSnapshots: ApplicationSnapshotConfigurationDescription.SnapshotsEnabled
    }'</code></pre> 
</div> 
<p><strong>Turn on both if they’re not already active:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">aws kinesisanalyticsv2 update-application \
    --application-name MyApplication \
    --current-application-version-id &lt;version-id&gt; \
    --application-configuration-update '{
        "ApplicationSystemRollbackConfigurationUpdate": {
            "RollbackEnabledUpdate": true
        },
        "ApplicationSnapshotConfigurationUpdate": {
            "SnapshotsEnabledUpdate": true
        }
    }'</code></pre> 
</div> 
<h3>Step 4: Take a manual snapshot (recommended)</h3> 
<p>Although the upgrade process takes an automatic snapshot, taking a manual snapshot gives you a named restore point that you can quickly identify.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws kinesisanalyticsv2 create-application-snapshot \
    --application-name MyApplication \
    --snapshot-name pre-flink-2.2-upgrade</code></pre> 
</div> 
<p>Verify that the snapshot is ready before proceeding:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws kinesisanalyticsv2 describe-application-snapshot \
    --application-name MyApplication \
    --snapshot-name pre-flink-2.2-upgrade</code></pre> 
</div> 
<p>Wait until&nbsp;SnapshotStatus&nbsp;is&nbsp;READY.</p> 
<h3>Step 5: Run the upgrade</h3> 
<p>Run the upgrade while the application is in&nbsp;RUNNING&nbsp;or&nbsp;READY&nbsp;(stopped) state. The following example upgrades a running application and points to the new JAR.</p> 
<p><strong>AWS CLI:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">aws kinesisanalyticsv2 update-application \
    --application-name MyApplication \
    --current-application-version-id &lt;version-id&gt; \
    --runtime-environment-update FLINK-2_2 \
    --application-configuration-update '{
        "ApplicationCodeConfigurationUpdate": {
            "CodeContentUpdate": {
                "S3ContentLocationUpdate": {
                    "FileKeyUpdate": "my-app-flink-2.2.jar"
                }
            }
        }
    }'</code></pre> 
</div> 
<p><strong>AWS Management Console:</strong></p> 
<p>To upgrade from the console, follow these steps:</p> 
<ol> 
 <li>Navigate to your application in the Amazon Managed Service for Apache Flink console.</li> 
 <li>Choose&nbsp;<strong>Configure</strong>.</li> 
 <li>Select the&nbsp;<strong>Flink 2.2</strong>&nbsp;runtime.</li> 
 <li>Point to your new application JAR on Amazon S3.</li> 
 <li>Select the snapshot to restore from (use&nbsp;<strong>Latest</strong>&nbsp;to start from the most recent snapshot).</li> 
 <li>Choose&nbsp;<strong>Update</strong>.</li> 
</ol> 
<p><strong>AWS CloudFormation:</strong></p> 
<p>Update the&nbsp;<code>RuntimeEnvironment</code>&nbsp;field in your template. AWS CloudFormation now performs an in-place update instead of deleting and recreating the application.</p> 
<p><strong>Terraform:</strong></p> 
<p>If you manage your Flink application with Terraform, you can perform the same in-place upgrade by updating the&nbsp;<code>runtime_environment</code> and code reference in your&nbsp;aws_kinesisanalyticsv2_application&nbsp;resource. Note: Terraform support for&nbsp;FLINK-2_2&nbsp;requires AWS provider version 6.40.0 or later (released April 8, 2026). Earlier provider versions don’t recognize this runtime value. First, update your provider version constraint:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "&gt;= 6.40.0"
    }
  }
}</code></pre> 
</div> 
<p>Then run&nbsp;terraform init -upgrade&nbsp;to pull the new provider.Next, update your application resource. Change&nbsp;<code>runtime_environment</code>&nbsp;from&nbsp;“FLINK-1_20”&nbsp;to&nbsp;“FLINK-2_2”&nbsp;and point to your new JAR:</p> 
<div class="hide-language"> 
 <pre><code class="lang-typescript">resource "aws_kinesisanalyticsv2_application" "my_app" {
  name                   = "MyApplication"
  runtime_environment    = "FLINK-2_2"
  service_execution_role = aws_iam_role.flink.arn
  application_configuration {
    application_code_configuration {
      code_content_type = "ZIPFILE"
      code_content {
        s3_content_location {
          bucket_arn = aws_s3_bucket.app_code.arn
          file_key   = "my-app-flink-2.2.jar"
        }
      }
    }
    application_snapshot_configuration {
      snapshots_enabled = true
    }
    flink_application_configuration {
      checkpoint_configuration {
        configuration_type = "DEFAULT"
      }
      monitoring_configuration {
        configuration_type = "CUSTOM"
        log_level          = "INFO"
        metrics_level      = "APPLICATION"
      }
      parallelism_configuration {
        auto_scaling_enabled = true
        configuration_type   = "CUSTOM"
        parallelism          = 4
        parallelism_per_kpu  = 1
      }
    }
  }
}</code></pre> 
</div> 
<p>Run the upgrade:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform plan    # Review the in-place update
terraform apply   # Apply the runtime change</code></pre> 
</div> 
<p>Terraform will perform an in-place update of the application, changing the runtime version and code location. The application will restart with the new Flink 2.2 runtime. To roll back with Terraform, revert <code>runtime_environment</code>&nbsp;to&nbsp;“FLINK-1_20”, point&nbsp;<code>file_key</code>&nbsp;back to your original JAR, and run&nbsp;terraform apply&nbsp;again. Note that you cannot restore a Flink 2.2 snapshot on Flink 1.x, so the rollback will start from the last Flink 1.x snapshot.</p> 
<p><strong>Important Terraform considerations:</strong></p> 
<ul> 
 <li>Auto-rollback and the <code>RollbackApplication</code> API aren’t directly exposed as Terraform resource attributes. If you need auto-rollback during the upgrade, enable it using the AWS CLI (Step 3) before running&nbsp;terraform apply, or use a provisioner/null_resource to call the CLI.</li> 
 <li>Always take a manual snapshot (Step 4) before running&nbsp;terraform apply&nbsp;for the upgrade. Terraform doesn’t automatically snapshot before updating the runtime.</li> 
</ul> 
<h3>Step 6: Monitor the upgrade</h3> 
<p>After initiating the upgrade, monitor the application to verify that it completes successfully.</p> 
<p><strong>Check application status:</strong></p> 
<p>The application should transition through&nbsp;RUNNING&nbsp;→&nbsp;UPDATING&nbsp;→&nbsp;RUNNING. Confirm the runtime version changed to 2.2:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws kinesisanalyticsv2 describe-application \
    --application-name MyApplication \
    --query 'ApplicationDetail.RuntimeEnvironment'</code></pre> 
</div> 
<p><strong>What to watch for:</strong></p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <thead> 
  <tr> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Scenario</strong></th> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>What happens</strong></th> 
   <th style="padding: 10px; border: 1px solid #dddddd;"><strong>Action</strong></th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Binary incompatibility</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Upgrade operation fails. Auto-rollback reverts to the previous version automatically.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Check operation logs for the exception, fix your code, and retry.</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">State incompatibility</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Upgrade appears to succeed but the application enters restart loops.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Monitor&nbsp;<code>numRestarts</code>&nbsp;metric. If restarts are continuous, invoke the Rollback API manually. Review the [State Compatibility Guide].</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Successful upgrade</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>numRestarts</code>&nbsp;is zero,&nbsp;uptime&nbsp;is increasing, checkpoints are completing.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Proceed to validation.</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Key CloudWatch metrics to monitor:</strong></p> 
<ol> 
 <li><code>numRestarts</code>: should be zero after upgrade</li> 
 <li><code>lastCheckpointDuration</code>: should be similar to pre-upgrade values</li> 
 <li><code>numberOfFailedCheckpoints</code>: should remain at zero</li> 
 <li><code>uptime</code>: should be steadily increasing</li> 
</ol> 
<h3>Step 7: Validate application behavior</h3> 
<p>After the application is running on Flink 2.2:</p> 
<ul> 
 <li>Confirm that data is being read from sources and written to sinks.</li> 
 <li>Compare the output with your pre-upgrade baseline.</li> 
 <li>Monitor latency, throughput, checkpoint duration, and resource utilization.</li> 
 <li>Run for at least 24 hours to confirm stable behavior: no memory leaks, no unexpected restarts, consistent checkpoint sizes.</li> 
</ul> 
<h3>Step 8: Rollback (if needed)</h3> 
<p>If the application is running but is unhealthy after the upgrade, invoke the Rollback API:</p> 
<p><strong>AWS CLI:</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws kinesisanalyticsv2 rollback-application \
    --application-name MyApplication \
    --current-application-version-id &lt;version-id&gt;</code></pre> 
</div> 
<p><strong>AWS Management Console:</strong></p> 
<ul> 
 <li>Navigate to your application.</li> 
 <li>Choose&nbsp;<strong>Actions</strong>, <strong>Roll back</strong>.</li> 
 <li>Confirm the rollback.</li> 
</ul> 
<p>During rollback, the application stops, reverts to the previous Flink version and application code, and restarts from the snapshot taken before the upgrade.</p> 
<p><strong>Important:</strong>&nbsp;You can’t restore a Flink 2.2 snapshot on Flink 1.x. Rollback uses the snapshot taken before the upgrade. This is why Steps 3 and 4 are critical.</p> 
<h2>Next steps</h2> 
<p>Your path depends on where you are today:</p> 
<ol> 
 <li><strong>If you’re new to Apache Flink:</strong>&nbsp;Start with the&nbsp;<a href="https://file+.vscode-resource.vscode-cdn.net/Users/fmorillo/Downloads/blog/link" rel="noopener noreferrer" target="_blank">guide to choosing the right API and language</a>, the&nbsp;<a href="https://docs.aws.amazon.com/managed-flink/latest/java/getting-started.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink getting started guide</a>, and the&nbsp;<a href="https://catalog.workshops.aws/managed-flink" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink workshop</a>.</li> 
 <li><strong>If you’re running Flink 1.x in production:</strong>&nbsp;Follow the migration steps in this post on a non-production replica first, then apply to production. For the complete reference, see the&nbsp;<a href="https://file+.vscode-resource.vscode-cdn.net/Users/fmorillo/Downloads/blog/link" rel="noopener noreferrer" target="_blank">Upgrading to Flink 2.2: Complete Guide</a>&nbsp;and the&nbsp;<a href="https://file+.vscode-resource.vscode-cdn.net/Users/fmorillo/Downloads/blog/link" rel="noopener noreferrer" target="_blank">State Compatibility Guide for Flink 2.2 Upgrades</a>.</li> 
 <li><strong>If you’re evaluating Flink 2.2 features:</strong>&nbsp;Launch a new application on the Flink 2.2 runtime to explore SQL/ML capabilities, the VARIANT data type, and the new join operators. See the&nbsp;<a href="https://github.com/aws-samples/amazon-managed-service-for-apache-flink-examples" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink sample applications on GitHub</a>&nbsp;for reference architectures.</li> 
 <li><strong>If you need help with your migration:</strong>&nbsp;Use the <a href="https://github.com/awslabs/managed-service-for-apache-flink-agent-steering-files" rel="noopener noreferrer" target="_blank">Kiro Power and Agent Skill for Amazon Managed Service for Apache Flink</a> to identify compatibility issues in your existing codebase and receive guidance on refactoring steps. You can also open a case through&nbsp;<a href="https://aws.amazon.com/support/" rel="noopener noreferrer" target="_blank">AWS Support</a>, post a question on&nbsp;<a href="https://repost.aws/tags/TAjj_AYVQYR-a2FMqOkMcEPg/amazon-managed-service-for-apache-flink" rel="noopener noreferrer" target="_blank">AWS re:Post for Amazon Managed Service for Apache Flink</a>, or reach out through the&nbsp;<a href="https://flink.apache.org/community/" rel="noopener noreferrer" target="_blank">Apache Flink community</a>.</li> 
</ol> 
<p>For the Apache Flink 2.2 documentation, see&nbsp;<a href="https://nightlies.apache.org/flink/flink-docs-release-2.2/" rel="noopener noreferrer" target="_blank">nightlies.apache.org/flink/flink-docs-release-2.2</a>. For Amazon Managed Service for Apache Flink documentation, see the&nbsp;<a href="https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html" rel="noopener noreferrer" target="_blank">Developer Guide</a>. For pricing, see the&nbsp;<a href="https://aws.amazon.com/managed-service-apache-flink/pricing/" rel="noopener noreferrer" target="_blank">pricing page</a>.</p> 
<h2>Conclusion</h2> 
<p>With Apache Flink 2.2 on Amazon Managed Service for Apache Flink, you get a modern Java 17 runtime, SQL-native AI/ML inference, improved state management performance, and a streamlined API surface. In-place upgrades with state preservation and auto-rollback make the migration straightforward. Test on a replica, follow the steps in this post, and start building on Flink 2.2.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-medium wp-image-90329" height="300" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/16/fmorillo-225x300.jpg" width="225" />
  </div> 
  <h3 class="lb-h4">Francisco Morillo</h3> 
  <p>Francisco Morillo&nbsp;is a Sr. Streaming Specialist Solutions Architect at AWS, helping customers design and operate real-time data processing applications using Amazon Managed Service for Apache Flink and Amazon Managed Streaming for Apache Kafka.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone wp-image-90607 size-full" height="1072" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/23/profilepic.jpg" width="940" />
  </div> 
  <h3 class="lb-h4">Mayank Juneja</h3> 
  <p>Mayank Juneja is a Senior Product Manager at AWS, leading Amazon Managed Service for Apache Flink. He lives at the intersection of real-time data streaming and AI, previously driving Flink SQL and AI inference products at Confluent.</p> 
 </div> 
</footer>
