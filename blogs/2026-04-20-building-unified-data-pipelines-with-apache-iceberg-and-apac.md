---
title: "Building unified data pipelines with Apache Iceberg and Apache Flink"
url: "https://aws.amazon.com/blogs/big-data/building-unified-data-pipelines-with-apache-iceberg-and-apache-flink/"
date: "Mon, 20 Apr 2026 16:59:46 +0000"
author: "Nikhil Jha"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>You can process real-time data from your data lake with <a href="https://docs.aws.amazon.com/managed-flink/" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink</a> without maintaining two separate pipelines. Yet many teams do exactly that, and the cost adds up fast. In this post, you build a unified pipeline using Apache Iceberg and Amazon Managed Service for Apache Flink that replaces the dual-pipeline approach. This walkthrough is for intermediate AWS users who are comfortable with <a href="https://docs.aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> and <a href="https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html" rel="noopener noreferrer" target="_blank">AWS Glue Data Catalog</a> but new to streaming from Apache Iceberg tables.</p> 
<h2>The dual-pipeline problem</h2> 
<p><img alt="Traditional dual-pipeline architecture with separate batch and streaming paths, each with its own ingestion, processing, storage, and serving layers, processing the same source data independently." class="wp-image-89997 size-full aligncenter" height="495" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-1-1.png" width="962" /></p> 
<p>This dual-pipeline approach creates three problems:</p> 
<ul> 
 <li><strong>Double the infrastructure costs.</strong> You run and pay for two separate compute environments, two storage layers, and two sets of monitoring. For example, if you’re spending $10,000/month on separate streaming and batch infrastructure, a meaningful portion of that spend is pure duplication.</li> 
 <li><strong>Data synchronization issues.</strong> Your batch and streaming consumers read from different copies of the data, processed at different times. When a transaction shows up in your real-time dashboard but not in your batch report (or vice versa), debugging the inconsistency takes hours.</li> 
 <li><strong>Operational complexity.</strong> Two pipelines mean two deployment processes, two failure modes to monitor, and two sets of schema evolution to manage. Your team spends time reconciling systems instead of building features.</li> 
</ul> 
<h2>Where this pattern fits</h2> 
<p>Before diving into the implementation, consider whether streaming from your data lake is the right approach for your use case.</p> 
<p><strong>Streaming from Apache Iceberg tables works well when</strong> you need data available within seconds to minutes and you query recent data frequently, multiple times per hour. Common scenarios include:</p> 
<ul> 
 <li><strong>Operational data stores</strong> — Stream customer profile updates to serve downstream applications like recommendation engines. When a customer updates their preferences, those changes reach your operational data store within seconds.</li> 
 <li><strong>Fraud detection</strong> — Stream transactions for immediate analysis. Start with a 3-second monitor interval and adjust based on your detection accuracy needs.</li> 
 <li><strong>Live dashboards</strong> — Power real-time analytics directly from your lake. This is the strongest starting point if you’re evaluating the approach for the first time, because the feedback loop is immediate and straightforward to validate.</li> 
 <li><strong>Event-driven architectures</strong> — Trigger downstream processes based on data changes in your Apache Iceberg tables.</li> 
</ul> 
<p><strong>Batch processing remains more cost-effective when</strong> you process data once per day or less, or you primarily query historical data. Batch queries on Apache Iceberg tables cost less because they don’t require a continuous Apache Flink runtime.</p> 
<h2>How Apache Iceberg solves this</h2> 
<p>Apache Iceberg’s snapshot-based architecture removes the need for a separate streaming pipeline. Think of snapshots like Git commits for your data. Each time you write data to your Iceberg table, Iceberg creates a new snapshot that points to the new data files while preserving references to existing files. Apache Flink reads only the changes between snapshots (the new files that arrived after the last checkpoint), rather than scanning the entire table. Atomicity, Consistency, Isolation, Durability (ACID) transactions prevent your concurrent reads and writes from producing partial or inconsistent results. For example, if your batch extract, transform, and load (ETL) job is writing 10,000 records while your Flink application is reading, ACID transactions mean that your streaming query sees either the complete batch of 10,000 records or none of them, not a partial set that could skew your analytics.</p> 
<p>The result is a single pipeline that handles both real-time and batch access from the same data, through the same storage layer, with the same schema.</p> 
<h2>Solution architecture</h2> 
<p>Your architecture uses four AWS services and one open source table format working together. The following diagram shows how these components connect, replacing the dual-pipeline pattern shown earlier with a single unified flow.</p> 
<p><img alt="Unified pipeline architecture with data flowing from Amazon S3 through Apache Iceberg tables, with AWS Glue Data Catalog managing metadata, and Amazon Managed Service for Apache Flink consuming incremental snapshots for near real-time processing." class="size-full wp-image-89963 aligncenter" height="581" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-2.png" width="1101" /></p> 
<p>Your source data lands in Amazon S3 as Apache Iceberg table files. AWS Glue Data Catalog tracks the metadata and schema. When new data arrives, Apache Iceberg creates a new snapshot that your application detects. Your Flink application monitors these snapshots and processes new records incrementally, reading only the files that arrived after the last checkpoint, not the entire table.</p> 
<p>You use four main components:</p> 
<ul> 
 <li><strong>Amazon S3</strong> — Foundational storage layer for your data lake</li> 
 <li><strong>Data Catalog</strong> — Metadata and schema management for Apache Iceberg tables</li> 
 <li><strong>Apache Iceberg</strong> — Table format with snapshot-based streaming capabilities</li> 
 <li><strong>Amazon Managed Service for Apache Flink</strong> — Stream processing and incremental consumption</li> 
</ul> 
<h2>Important notices</h2> 
<p>Before implementing this solution, evaluate these risks for your environment:</p> 
<ul> 
 <li><strong>Data security:</strong> Streaming from data lakes exposes data to additional processing systems. Classify your data before implementation—customer profile updates and transaction data typically contain personally identifiable information (PII) and treat them as confidential. Apply encryption at rest and in transit for confidential data. Key risks include unauthorized data access through misconfigured Amazon S3 bucket policies or overly permissive IAM roles. Mitigations: use the resource-scoped IAM policy and TLS-enforcing bucket policy provided in the Security section.</li> 
 <li><strong>Data integrity:</strong> Misconfigured checkpoints or schema changes during streaming can lead to data inconsistency. Mitigations: enable exactly-once processing semantics and test schema evolution in a non-production environment first.</li> 
 <li><strong>Compliance:</strong> Verify that real-time data processing meets your regulatory requirements. For workloads subject to HIPAA, confirm that you use HIPAA Eligible Services and have a Business Associate Agreement (BAA) with AWS. For PCI-DSS or GDPR workloads, review the relevant compliance documentation on the AWS Compliance page. Implement data retention policies that comply with your regulatory framework.</li> 
 <li><strong>Cost:</strong> Nearly continuous streaming incurs ongoing compute costs. Monitor usage to avoid unexpected charges. Cost estimates in this post are based on pricing as of March 2026 and might change. Verify current pricing on the relevant AWS service pricing pages.</li> 
 <li><strong>Operational:</strong> Pipeline failures might impact downstream systems. Implement monitoring and alerting before running in production.</li> 
</ul> 
<h2>Prerequisites</h2> 
<p>Before you begin, make sure that you have the following in place. This walkthrough assumes intermediate Python skills (comfortable with functions, error handling, and environment variables), basic Apache Flink concepts (streaming compared to batch processing), and basic <a href="https://docs.aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (AWS IAM)</a> knowledge (creating roles and attaching policies). Plan for approximately 90–120 minutes, including setup, implementation, and testing. First-time setup might take longer as you download dependencies and configure AWS resources. Expected AWS costs: approximately $5–10 if you complete the walkthrough within 2 hours and clean up resources immediately afterward. The primary cost driver is Amazon Managed Service for Apache Flink runtime ($0.11/hour per Kinesis Processing Unit (KPU)). You can minimize costs by stopping your application when not in use.</p> 
<ul> 
 <li>An AWS account with AWS IAM permissions for: <code>s3:GetObject</code>, <code>s3:PutObject</code>, <code>s3:ListBucket</code> on your data bucket; <code>glue:GetDatabase</code>, <code>glue:GetTable</code> for catalog access; and <code>flink:CreateApplication</code>, <code>flink:StartApplication</code> for Amazon Managed Service for Apache Flink</li> 
 <li>An existing Amazon S3 bucket for your data lake</li> 
 <li>An AWS Glue Data Catalog database configured</li> 
 <li>Apache Flink 1.19.1 installed locally</li> 
 <li>Python 3.8 or later</li> 
 <li>Java 11 or a more recent version</li> 
 <li><a href="https://docs.aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> configured with credentials (aws configure)</li> 
</ul> 
<h3>Required Java Archive (JAR) dependencies</h3> 
<p>You need multiple JAR files because your Flink application coordinates between different systems—Amazon S3 for storage, AWS Glue for metadata, Hadoop for file operations, and Apache Iceberg for the table format. Each JAR handles a specific part of this integration. Missing even one causes ClassNotFoundException errors at runtime.</p> 
<ul> 
 <li>iceberg-flink-runtime-1.19-1.6.1.jar — Core Apache Iceberg integration with Apache Flink</li> 
 <li>iceberg-aws-bundle-1.6.1.jar — AWS-specific Apache Iceberg functionality for Amazon S3 and AWS Glue</li> 
 <li>flink-s3-fs-hadoop-1.19.1.jar — Provides Apache Flink read and write access to Amazon S3</li> 
 <li>flink-sql-connector-hive-3.1.3_2.12-1.19.1.jar — Hive metastore connector for catalog compatibility</li> 
 <li>hadoop-common-3.4.0.jar — Core Hadoop libraries required by Apache Iceberg</li> 
 <li>flink-shaded-hadoop-2-uber-2.8.3-10.0.jar — Repackaged Hadoop dependencies that avoid version conflicts with Apache Flink</li> 
 <li>hadoop-hdfs-client-3.4.0.jar — Hadoop Distributed File System (HDFS) client libraries for file system operations</li> 
 <li>flink-json-1.19.1.jar — JSON format support for Apache Flink</li> 
 <li>hadoop-aws-3.4.0.jar — Hadoop integration with AWS services</li> 
 <li>hadoop-client-3.4.0.jar — Hadoop client libraries</li> 
 <li>aws-java-sdk-bundle-1.12.261.jar — AWS SDK for authentication and service access</li> 
</ul> 
<div class="hide-language"> 
 <pre><code class="lang-code">jars = [
    "flink-s3-fs-hadoop-1.19.1.jar",
    "flink-sql-connector-hive-3.1.3_2.12-1.19.1.jar",
    "hadoop-common-3.4.0.jar",
    "flink-shaded-hadoop-2-uber-2.8.3-10.0.jar",
    "iceberg-flink-runtime-1.19-1.6.1.jar",
    "iceberg-aws-bundle-1.6.1.jar",
    "hadoop-hdfs-client-3.4.0.jar",
    "flink-json-1.19.1.jar",
    "hadoop-aws-3.4.0.jar",
    "hadoop-client-3.4.0.jar",
    "aws-java-sdk-bundle-1.12.261.jar"
]</code></pre> 
</div> 
<h2>Technical implementation</h2> 
<p>The sample code in this post is available under the MIT-0 license.This section walks you through building the streaming pipeline step by step. You create a single Python file, iceberg_streaming.py, with three functions that run in sequence. Your main() function calls them in order: set up the Apache Flink environment, register the Data Catalog, then start the streaming query.</p> 
<h3>Set up your Apache Flink environment</h3> 
<p>To prepare your Apache Flink environment:</p> 
<ol> 
 <li>Download the required JAR files listed in the prerequisites section.</li> 
 <li>Place the JAR files in a lib directory in your project folder.</li> 
 <li>Configure your <code>HADOOP_CLASSPATH</code> environment variable to point to the lib directory.</li> 
 <li>Create your streaming execution environment by adding the following function to <code>iceberg_streaming.py</code>:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-python">def setup_environment():
    """Configure the Flink streaming runtime."""
    try:
        os.environ['HADOOP_CLASSPATH'] = os.path.join(os.getcwd(), 'lib', '*')
        env = StreamExecutionEnvironment.get_execution_environment()
        env.set_parallelism(1)
        settings = EnvironmentSettings.new_instance().in_streaming_mode().build()
        t_env = StreamTableEnvironment.create(env, settings)
        return t_env
    except Exception as e:
        print(f"Failed to initialize Flink environment: {e}")
        raise</code></pre> 
</div> 
<ol start="5"> 
 <li>Verify your environment by running flink –version. If the command isn’t found, confirm that Apache Flink 1.19.1 is installed and that your PATH includes the Flink bin directory.</li> 
</ol> 
<h3>Configure AWS Glue Data Catalog</h3> 
<p>To connect your Flink application to Data Catalog:</p> 
<ol> 
 <li>Open your <code>iceberg_streaming.py</code> file.</li> 
 <li>Add the <code>create_iceberg_source()</code> function shown in the following section.</li> 
 <li>Replace the placeholder values with your actual AWS resources before running. These values are static configuration strings, not user input — do not construct them from external or untrusted sources at runtime.</li> 
 <li>Save the file.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-python">def create_iceberg_source(t_env):
    """Register the AWS Glue Data Catalog as an Iceberg catalog."""
    try:
        catalog_sql = """
        CREATE CATALOG glue_catalog WITH (
            'type'='iceberg',
            'catalog-impl'='org.apache.iceberg.aws.glue.GlueCatalog',
            'warehouse'='s3://&lt;example-data-lake-bucket&gt;',
            'io-impl'='org.apache.iceberg.aws.s3.S3FileIO',
            'aws.region'='us-east-1',
            'hadoop-conf.fs.s3a.aws.credentials.provider'=
                'com.amazonaws.auth.DefaultAWSCredentialsProviderChain',
            'hadoop-conf.fs.s3a.endpoint'='s3.amazonaws.com',
            'property-version'='1'
        )
        """
        t_env.execute_sql(catalog_sql)
        t_env.use_catalog("glue_catalog")
        t_env.use_database("streaming_db")
    except Exception as e:
        print(f"Failed to configure Iceberg catalog: {e}")
        raise</code></pre> 
</div> 
<h3>Set up streaming logic</h3> 
<p>This function configures Apache Flink to monitor your Apache Iceberg table continuously and process new records as they arrive. Checkpointing runs every 10 seconds to track progress—if the job restarts, it resumes from the last checkpoint rather than reprocessing the entire table.Notice the monitor-interval parameter, it controls how frequently Apache Flink checks for new Apache Iceberg snapshots. A 3-second interval provides near real-time processing but generates approximately 1,200 Amazon S3 LIST API calls per hour (at $0.005 per 1,000 requests, roughly $0.04/month per table based on pricing as of March 2026). For less time-sensitive workloads, increase this to 30s to reduce API costs by 90%.Replace <code>customer_events</code> with the name of your Apache Iceberg table in Data Catalog:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def process_record(row):
    """Validate and process each record from the stream."""
    try:
        if row is None:
            raise ValueError("Received null row")
        required_fields = ["event_type", "timestamp"]
        for field in required_fields:
            if field not in row:
                raise ValueError(f"Missing required field: {field}")
        # Validate field types and content
        if not isinstance(row.get("event_type"), str) or len(row["event_type"]) &gt; 256:
            raise ValueError("event_type must be a string under 256 characters")
        if not isinstance(row.get("timestamp"), (str, int)):
            raise ValueError("timestamp must be a string or integer")
        # Replace with your business logic
        print(f"Processing record: {row}")
    except ValueError as e:
        print(f"Validation error for record {row}: {e}")
    except Exception as e:
        print(f"Error processing record {row}: {e}")
def stream_data(t_env):
    """Start the streaming query and process results."""
    try:
        configuration = t_env.get_config().get_configuration()
        configuration.set_string("table.dynamic-table-options.enabled", "true")
        configuration.set_string("execution.checkpointing.interval", "10000")
        query = """
        SELECT * FROM customer_events /*+ OPTIONS(
            'streaming'='true',
            'monitor-interval'='3s',
            'table.exec.iceberg.cell-based-snapshot'='true'
        ) */
        """
        table_result = t_env.execute_sql(query)
        with table_result.collect() as results:
            for row in results:
                process_record(row)
    except Exception as e:
        print(f"Streaming query failed: {e}")
        raise</code></pre> 
</div> 
<h3>Putting it together</h3> 
<p>Your <code>main()</code> function calls the three steps in order:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def main():
    try:
        t_env = setup_environment()
        create_iceberg_source(t_env)
        stream_data(t_env)
    except Exception as e:
        print(f"Pipeline failed: {e}")
        raise
if __name__ == "__main__":
    main()</code></pre> 
</div> 
<p>Run the pipeline locally:<code>python iceberg_streaming.py</code>Package the application and submit it to Amazon Managed Service for Apache Flink using the console or the AWS Command Line Interface (AWS CLI).</p> 
<h2>Running in production</h2> 
<p>Moving from a local test to a production deployment requires tuning four areas: performance, monitoring, cost, and security. This section covers the key decisions for each.</p> 
<h3>Performance tuning</h3> 
<p>Determine your latency requirements before tuning. For fraud detection, you need subsecond processing. For daily reporting dashboards, you can tolerate minutes of delay.</p> 
<p><strong>Partition pruning</strong> reduces the amount of data scanned per query. Proper partitioning can significantly reduce query times for time series data partitioned by date. To implement, create your Apache Iceberg table with partition columns (<code>PARTITIONED BY (date_column) in your CREATE TABLE statement</code>), then include partition filters in your <code>WHERE clause: WHERE date_column &gt;= CURRENT_DATE - INTERVAL '7' DAY</code>.</p> 
<p><strong>Parallel processing</strong> matches your data volume and throughput requirements. For most workloads under 10,000 records per second, a parallelism of 1–4 is sufficient. Scale up incrementally and monitor backpressure metrics (indicators that data arrives faster than your pipeline processes it, causing queuing) to find the right setting.</p> 
<p><strong>Checkpoint tuning</strong> balances reliability and latency. Consider how much data you can afford to reprocess after a failure. If you process 1,000 records per second with 10-second checkpoints, a failure means reprocessing up to 10,000 records. When that’s acceptable, 10 seconds works well. For faster recovery or higher volumes, reduce to 5 seconds.</p> 
<p><strong>Resource allocation</strong> — Right-size your Apache Flink cluster to avoid over-provisioning. Monitor CPU and memory utilization during your initial runs and adjust task manager resources accordingly.</p> 
<h3>Monitoring</h3> 
<p>Configure your production deployment with the following checkpoint settings. These work well for moderate data volumes (up to 10,000 records per second), providing exactly-once processing semantics. This means that the pipeline processes each record exactly once, even if your application restarts. Adjust the checkpoint interval based on your latency requirements. Add this to your setup_environment() function after creating the table environment.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">config_dict = {
    "execution.checkpointing.interval": "30000",
    "execution.checkpointing.mode": "EXACTLY_ONCE",
    "execution.checkpointing.timeout": "600000",
    "state.backend": "filesystem",
    "state.checkpoints.dir": "s3://&lt;example-data-lake-bucket&gt;/checkpoints"
}</code></pre> 
</div> 
<p>Use <a href="https://docs.aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> to track checkpoint duration, records processed per second, and backpressure metrics. A 10-second checkpoint interval means writing state to Amazon S3 360 times per hour. For a 1 MB state size, that’s approximately 8.6 GB per day in checkpoint storage—at Amazon S3 Standard pricing of $0.023/GB, roughly $0.20/day or $6/month per application based on current pricing. If the checkpoint duration exceeds 50% of your interval, increase the interval or add parallelism.</p> 
<h3>Cost management</h3> 
<p>Use <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/intelligent-tiering.html" rel="noopener noreferrer" target="_blank">Amazon S3 Intelligent-Tiering</a> for your Apache Iceberg data files, which typically have predictable access patterns after initial processing. Configure Apache Iceberg’s table expiration to automatically clean up early snapshots. This can reduce storage costs by an estimated 20–30%, though your results vary depending on write frequency and retention policies.</p> 
<p>Right-size your Apache Flink resources based on actual throughput needs. Start with a minimal configuration and scale up based on observed backpressure and checkpoint duration metrics. Use <a href="https://docs.aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> Spot Instances where workload interruptions are acceptable, for example, in development and testing environments.</p> 
<p>Set data retention policies on both your Apache Iceberg tables and checkpoint storage to avoid storing data longer than necessary.</p> 
<h3>Security</h3> 
<p>Security is a <a href="https://aws.amazon.com/compliance/shared-responsibility-model/" rel="noopener noreferrer" target="_blank">shared responsibility</a> between you and AWS. AWS is responsible for the security of the cloud, including the hardware, software, networking, and facilities that run AWS services. You are responsible for security in the cloud, configuring access controls, encrypting data, and managing your application security. Apply these controls in priority order.</p> 
<p><strong>AWS IAM roles</strong> — Use AWS IAM roles with least-privilege access, scoped to specific resources. The following example policy restricts permissions to your data lake bucket and AWS Glue catalog:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::&lt;example-data-lake-bucket&gt;/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::&lt;example-data-lake-bucket&gt;",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "&lt;your-vpc-endpoint-id&gt;"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["glue:GetDatabase", "glue:GetTable"],
      "Resource": [
        "arn:aws:glue:us-east-1:&lt;account-id&gt;:catalog",
        "arn:aws:glue:us-east-1:&lt;account-id&gt;:database/streaming_db",
        "arn:aws:glue:us-east-1:&lt;account-id&gt;:table/streaming_db/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:us-east-1:&lt;account-id&gt;:key/&lt;your-kms-key-id&gt;"
    }
  ]
}</code></pre> 
</div> 
<p>Scoping permissions to specific Amazon S3 buckets, AWS Glue databases, and AWS Key Management Service (AWS KMS) keys restrict access to only the resources your pipeline requires. Review IAM policies quarterly using the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html" rel="noopener noreferrer" target="_blank">IAM Access Analyzer</a> to identify and remove unused permissions.</p> 
<p><strong>Encryption</strong> — Configure server-side encryption with <a href="https://docs.aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS Key Management Service (AWS KMS)</a> customer managed keys (SSE-KMS) for your Amazon S3 buckets. Using customer managed keys requires additional review from your security team. Confirm your key management policies, rotation procedures, and access controls before implementation. Enable automatic key rotation annually. For encryption in transit, enforce TLS by adding a bucket policy that denies non-HTTPS access:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
  "Effect": "Deny",
  "Principal": "*",
  "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::&lt;example-data-lake-bucket&gt;/*",
    "arn:aws:s3:::&lt;example-data-lake-bucket&gt;"
  ],
  "Condition": {
    "Bool": { "aws:SecureTransport": "false" }
  }
}</code></pre> 
</div> 
<p><strong>Amazon S3 bucket hardening</strong> — Enable Block Public Access on your buckets to prevent accidental public exposure:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws s3api put-public-access-block \
  --bucket &lt;example-data-lake-bucket&gt; \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true</code></pre> 
</div> 
<p>Enable versioning on buckets that store critical data and checkpoints to protect against accidental deletion. For production environments with sensitive data, consider enabling MFA Delete on versioned buckets. Enable S3 server access logging to track requests for security auditing.</p> 
<p><a href="https://aws.amazon.com/vpc/" rel="noopener noreferrer" target="_blank"><strong>Amazon Virtual Private Cloud (Amazon VPC)</strong></a> –Use <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html" rel="noopener noreferrer" target="_blank">Amazon VPC endpoints</a> for private communication between your Apache Flink cluster and AWS services, removing public internet routing by keeping traffic within the AWS network.</p> 
<p><strong>Access logging</strong> – Enable <a href="https://docs.aws.amazon.com/cloudtrail/" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a> data events to log Amazon S3 object-level API calls (GetObject, PutObject) and Data Catalog API calls. Store logs in a separate Amazon S3 bucket with restricted access and enable log file integrity validation. Run regular compliance checks using <a href="https://docs.aws.amazon.com/config/" rel="noopener noreferrer" target="_blank">AWS Config</a>.</p> 
<h3>Operational practices</h3> 
<p>Set up a continuous integration and continuous deployment (CI/CD) pipeline to automate deployment and testing. Use version control to track schema and code changes. With Apache Iceberg’s schema evolution support, you can add columns without rewriting existing data files. Establish rollback procedures using Apache Iceberg’s snapshot-based architecture, so you can roll back to a previous table state if a bad write corrupts your data.</p> 
<h2>Troubleshooting</h2> 
<p>If you run into issues during setup or execution, use the following table to diagnose common errors.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Error</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Cause</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Solution</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ClassNotFoundException</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Missing JAR files</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Check the dependencies in your lib directory and confirm <code>HADOOP_CLASSPATH</code> points to the correct path</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Table not found</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Database name mismatch</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Check that the database name in <code>t_env.use_database()</code> matches the AWS Glue database where you registered your table</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Checkpoint failures</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon S3 permissions</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Check that your Amazon S3 bucket policy grants <code>s3:PutObject</code> for the checkpoint location</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">AWS credential errors</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Missing AWS IAM configuration</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Check that the AWS IAM role attached to your Apache Flink application has <code>glue:GetTable</code>, <code>glue:GetDatabase</code>, and <code>s3:GetObject</code> permissions on the relevant resources</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Snapshot not found</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Table modified during query</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Increase monitor-interval or implement retry logic in your <code>process_record()</code> function</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Schema mismatch</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Table schema changed between snapshots</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Review Apache Iceberg schema evolution settings and confirm backward compatibility</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Clean up</h2> 
<p>To avoid ongoing charges, delete the resources that you created during this walkthrough.</p> 
<ol> 
 <li>Stop your Amazon Managed Service for Apache Flink application. Open the <a href="https://console.aws.amazon.com/flink/" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink console</a>, choose your application name, choose <strong>Stop</strong>, and confirm the action. Or use the AWS CLI:</li> 
</ol> 
<p><code>aws kinesisanalyticsv2 stop-application --application-name your-app-name</code></p> 
<ol start="2"> 
 <li>Delete the Amazon S3 buckets that you created for data storage and checkpoints. For instructions, see <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-bucket.html" rel="noopener noreferrer" target="_blank">Deleting a bucket</a> in the Amazon S3 User Guide.</li> 
 <li>Remove the Apache Iceberg tables from your <a href="https://docs.aws.amazon.com/glue/latest/dg/console-tables.html" rel="noopener noreferrer" target="_blank">Data Catalog</a>.</li> 
 <li>Delete the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_manage_delete.html" rel="noopener noreferrer" target="_blank">AWS IAM roles and policies</a> created specifically for this walkthrough.</li> 
 <li>If you created an Amazon VPC or Amazon VPC endpoints for testing, <a href="https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html" rel="noopener noreferrer" target="_blank">delete those resources</a>.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>Maintaining separate streaming and batch pipelines doubles your infrastructure costs, creates data synchronization issues, and adds operational complexity that slows your team down. In this post, you replaced that dual-pipeline architecture with a single system built on Apache Iceberg and Amazon Managed Service for Apache Flink. You configured a Flink environment with the required JAR dependencies, connected it to Data Catalog, and implemented streaming queries that read new records incrementally with exactly-once processing semantics. The same data, the same storage layer, the same schema—accessible to both your real-time and batch consumers.</p> 
<p>To extend this solution, try these next steps based on your use case:</p> 
<ul> 
 <li><strong>If you’re processing high volumes (&gt;10,000 records/sec):</strong> Start with partition pruning. Add PARTITIONED BY (date_column) to your table definition, this typically reduces query times by 60–80%.</li> 
 <li><strong>If you need production monitoring:</strong> Implement custom <a href="https://docs.aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> metrics. Track checkpoint duration, records processed per second, and backpressure to catch issues before they impact your pipeline.</li> 
 <li><strong>If you have variable workloads:</strong> Configure auto scaling for your Apache Flink cluster. See the <a href="https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink Developer Guide</a> for detailed guidance.</li> 
</ul> 
<p>Share your implementation experience in the comments, your use case, data volumes, latency improvements, and cost reductions help other readers calibrate their expectations. To get started, try the <a href="https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Apache Flink Developer Guide</a> and the <a href="https://iceberg.apache.org/docs/latest/" rel="noopener noreferrer" target="_blank">Apache Iceberg documentation</a> on the Apache Iceberg website.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Headshot of Nikhil" class="alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-3.jpeg" width="100" />
  </div> 
  <h3 class="lb-h4">Nikhil Jha</h3> 
  <p><strong>Nikhil Jha</strong>&nbsp;is a Principal Delivery Consultant at AWS Professional Services, helping enterprises navigate complex modernization journeys. He builds data and AI solutions for AWS customers. Outside of work he likes swimming and hiking.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Headshot of Vyas" class="alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-4-269x300.png" width="100" />
  </div> 
  <h3 class="lb-h4">Vyas Garigipati</h3> 
  <p><strong>Vyas Garigipati</strong>&nbsp;is a Delivery Consultant at AWS Professional Services, with experience building scalable, distributed systems. He specializes in designing and building AI-powered, high-availability, multi-region architectures and helps customers deploy resilient, production ready solutions on AWS.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Headshot of Vafa" class="alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-5.jpeg" width="100" />
  </div> 
  <h3 class="lb-h4">Vafa Ahmadiyeh</h3> 
  <p><strong>Vafa Ahmadiyeh</strong>&nbsp;is a Principal Lead Technologist at AWS, specializing in cloud architecture for the global financial services sector. He partners with major financial institutions to modernize their infrastructure and accelerate their migration to AWS, with a focus on building secure, scalable distributed systems and platforms designed for highly regulated environments.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Headshot of Kaushal" class="alignleft" height="100" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/BDB-5291-image-6.png" width="100" />
  </div> 
  <h3 class="lb-h4">Kaushal (KK) Agrawal</h3> 
  <p><strong>Kaushal (KK) Agrawal </strong>is a Principal Technology Delivery Leader for the Digital Native Segment of AWS Professional Services, working with top-tier customers to deliver innovation at the intersection of AI and Cloud.</p> 
 </div> 
</footer>
