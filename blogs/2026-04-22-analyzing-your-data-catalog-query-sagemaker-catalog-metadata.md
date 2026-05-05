---
title: "Analyzing your data catalog: Query SageMaker Catalog metadata with SQL"
url: "https://aws.amazon.com/blogs/big-data/analyzing-your-data-catalog-query-sagemaker-catalog-metadata-with-sql/"
date: "Wed, 22 Apr 2026 15:37:38 +0000"
author: "Ramesh H Singh"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>As your data and machine learning (ML) assets grow, tracking which assets lack documentation or monitoring asset registration trends becomes challenging without custom reporting infrastructure. You need visibility into your catalog’s health, without the overhead of managing ETL jobs. The metadata feature of <a href="https://aws.amazon.com/sagemaker/" rel="noopener noreferrer" target="_blank">Amazon SageMaker</a> provides this capability to users.&nbsp;Converting catalog asset metadata into <a href="https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/introduction.html" rel="noopener noreferrer" target="_blank">Apache Iceberg</a> tables stored in <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html" rel="noopener noreferrer" target="_blank">Amazon S3 Tables</a> removes the need to build and maintain custom ETL pipelines. Your team can then query asset metadata directly using standard SQL tools. You can now answer governance questions like asset registration trends, classification status, and metadata completeness using standard SQL queries through tools like <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a>, <a href="https://aws.amazon.com/sagemaker/unified-studio/" rel="noopener noreferrer" target="_blank">Amazon SageMaker Unified Studio</a> notebooks, and BIsystems.</p> 
<p>This automated approach reduces ETL development time and gives your team visibility into catalog health, compliance gaps, and asset lifecycle patterns. The exported tables include technical metadata, business metadata, project ownership details, and timestamps, partitioned by snapshot date to enable time travel queries and historical analysis. Teams can use this capability to proactively monitor catalog health, identify gaps in documentation, track asset lifecycle patterns, and make sure that governance policies are consistently applied.</p> 
<h2>How metadata export works</h2> 
<p>After you enable the metadata export feature, it runs automatically on a daily schedule:</p> 
<ol> 
 <li><strong>SageMaker Catalog creates the infrastructure</strong> — An Amazon Simple Storage Service (Amazon S3) table bucket named <code>aws-sagemaker-catalog</code> is created with an <code>asset_metadata</code> namespace and an empty asset table.</li> 
 <li><strong>Daily snapshots are captured</strong> — A scheduled job runs once per day around midnight (local time per AWS Region) to export updated asset metadata.</li> 
 <li><strong>Metadata is structured and partitioned</strong> — The export captures technical metadata (resource_id, resource_type), business metadata (asset_name, business_description), project ownership details, and timestamps, partitioned by <code>snapshot_date</code> for query performance.</li> 
 <li><strong>Data becomes queryable</strong> — Within 24 hours, the asset table appears in Amazon SageMaker Unified Studio under the <code>aws-sagemaker-catalog</code> bucket and becomes accessible through Amazon Athena, Studio notebooks, or external BI tools.</li> 
 <li><strong>Teams query using standard SQL</strong> — Data teams can now answer questions like “How many assets were registered last month?” or “Which assets lack business descriptions?” without building custom ETL pipelines.</li> 
</ol> 
<p>The export evaluates catalog assets and their metadata properties in the domain, converting them into Apache Iceberg table format. The data flows into downstream analytics operations immediately, with no separate ETL or batch processes to maintain. The exported metadata becomes part of a queryable data lake that supports time-travel queries and historical analysis.</p> 
<p>In this post, we demonstrate how to use the metadata export capability in Amazon SageMaker Catalog and perform analytics on these tables. We explore the following specific use-cases.</p> 
<ul> 
 <li>Audit historical changes to investigate what an asset looked like at a specific point in time.</li> 
 <li>Monitor asset growth view how the data catalog has grown over the last 30 days.</li> 
 <li>Track metadata improvements to see which assets gained descriptions or ownership over time.</li> 
</ul> 
<h2>Solution overview</h2> 
<div class="wp-caption alignleft" id="attachment_90416" style="width: 1431px;">
 <img alt="AWS Cloud architecture diagram showing data pipeline from Amazon SageMaker Catalog to Amazon S3 Tables with daily export, connecting to query engines including Amazon Athena, Amazon Redshift, and Apache Spark" class="wp-image-90416 size-full" height="801" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-1.jpeg" width="1421" />
 <p class="wp-caption-text" id="caption-attachment-90416">Figure 1 – SageMaker catalog export to S3 Tables</p>
</div> 
<p>The architecture consists of three key components:</p> 
<ol> 
 <li>Amazon SageMaker Catalog exports asset metadata daily to Amazon S3.</li> 
 <li>S3 Tables stores metadata as Apache Iceberg tables in the <code>aws-sagemaker-catalog</code> bucket with ACID compliance and time travel.</li> 
 <li>Query engines (Amazon Athena, <a href="https://aws.amazon.com/pm/redshift/" rel="noopener noreferrer" target="_blank">Amazon Redshift</a>, and Apache Spark) access metadata using standard SQL from the <code>asset_metadata.asset</code> table.</li> 
</ol> 
<h3>What metadata is exposed?</h3> 
<p>SageMaker Catalog exports metadata in the <code>asset_metadata.asset</code> table:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Metadata Type</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Fields</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Technical metadata</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>resource_id</code>, <code>resource_type_enum</code>, <code>account_id</code>, <code>region</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Resource identifiers (ARN), types (<code>GlueTable</code>, <code>RedshiftTable</code>, <code>S3Collection</code>), and location</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Namespace hierarchy</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>catalog</code>, <code>namespace</code>, <code>resource_name</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Organizational structure for assets</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Business metadata</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>asset_name</code>, <code>business_description</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Human-readable names and descriptions</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Ownership</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>extended_metadata['owningEntityId']</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Asset ownership information</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Timestamps</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>asset_created_time</code>, <code>asset_updated_time</code>, <code>snapshot_time</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Creation</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Custom metadata</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>extended_metadata['form-name.field-name']</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">User-defined metadata forms as key-value pairs</td> 
  </tr> 
 </tbody> 
</table> 
<p>The <code>snapshot_time</code> column supports point-in-time analysis and query of historical catalog states.</p> 
<h2>Prerequisites</h2> 
<p>To follow along with this post, you must have the following:</p> 
<ul> 
 <li>An <a href="https://aws.amazon.com/sagemaker/unified-studio/" rel="noopener noreferrer" target="_blank">Amazon SageMaker Unified Studio</a> domain set up with a domain owner or domain unit owner permissions. 
  <ul> 
   <li>A SageMaker Unified Studio domain identifier</li> 
  </ul> </li> 
 <li><a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> permissions for configuring metadata export.</li> 
 <li>Grant catalog, database, and table Select and Describe permissions with <a href="https://aws.amazon.com/lake-formation/" rel="noopener noreferrer" target="_blank">AWS Lake Formation</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> version 2.33.0 or later installed and configured</li> 
 <li>An Amazon SageMaker project for publishing assets.</li> 
</ul> 
<p>For SageMaker Unified Studio domain setup instructions, refer to the SageMaker Unified Studio <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/getting-started.html" rel="noopener noreferrer" target="_blank">Getting started</a> guide.</p> 
<p>After you complete the prerequisites, complete the following steps.</p> 
<ol> 
 <li>Add this policy to our IAM user or role to enable metadata export. If using SageMaker Unified Studio to query the catalog, add this policy to the <code>AmazonSageMakerAdminIAMExecutionRole</code> managed role.</li> 
</ol> 
<pre><code class="lang-json">{ "Version": "2012-10-17", 
"Statement": [ 
{
 "Effect": "Allow",
 "Action": [ "datazone:GetDataExportConfiguration",
 "datazone:PutDataExportConfiguration"
 ],
 "Resource": "*"
 },
 {
 "Effect": "Allow",
 "Action": [
 "s3tables:CreateTableBucket",
 "s3tables:PutTableBucketPolicy"
 ],
 "Resource": "arn:aws:s3tables:*:*:bucket/aws-sagemaker-catalog" 
} 
]
}</code></pre> 
<ol start="2"> 
 <li><strong>Grant describe</strong> and <strong>select</strong> permissions for SageMaker Catalog with AWS Lake Formation. This step can be performed in the AWS Lake Formation console. 
  <ol type="a"> 
   <li>Select <strong>Permissions</strong> -&gt; <strong>Data permissions</strong> and choose <strong><strong>Grant.<br /> </strong></strong><p></p> <p></p>
    <div class="wp-caption alignnone" id="attachment_90415" style="width: 1435px;">
     <img alt="AWS Lake Formation Grant Permissions interface showing principal type selection with IAM users and roles option selected and AmazonSageMakerAdminIAMExecutionRole assigned" class="size-full wp-image-90415" height="878" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-2.jpeg" width="1425" />
     <p class="wp-caption-text" id="caption-attachment-90415">Figure 2 – AWS Lake Formation grant permission</p>
    </div></li> 
   <li>Under <strong>Principal type</strong>, select <strong>Principals</strong>, <strong>IAM users and roles</strong> and the AWS managed <strong>AmazonSageMakerAdminIAMExecutionRole</strong> execution role.</li> 
   <li>Choose <strong>Named Data Catalog resources</strong>.</li> 
   <li>Under <strong>Catalogs</strong>, search for and select <strong>&lt;account-id&gt;:s3tablecatalog/aws-sagemaker-catalog.</strong></li> 
   <li>Under <strong>Databases</strong>, select <strong>asset_metadata</strong> database. 
    <div class="wp-caption alignnone" id="attachment_90414" style="width: 1439px;">
     <img alt="AWS Lake Formation Grant Permissions page showing Named Data Catalog resources method with s3tablescatalog/aws-sagemaker-catalog selected, asset_metadata database, and asset table configured" class="size-full wp-image-90414" height="1073" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-3.jpeg" width="1429" />
     <p class="wp-caption-text" id="caption-attachment-90414">Figure 3 – AWS Lake Formation catalog, database, and table</p>
    </div> <p></p>
    <div class="wp-caption alignnone" id="attachment_90413" style="width: 1438px;">
     <img alt="AWS Lake Formation Grant Permissions interface showing table permissions with Select and Describe checked, grantable permissions section, and All data access radio button selected" class="size-full wp-image-90413" height="1247" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-4.jpeg" width="1428" />
     <p class="wp-caption-text" id="caption-attachment-90413">Figure 4 – AWS Lake Formation grant permission</p>
    </div></li> 
   <li>For <strong>Table</strong>, select <strong>asset</strong>.</li> 
   <li>Under <strong>Table permissions</strong>, check <strong>Select</strong> and <strong>Describe.</strong></li> 
   <li>Choose <strong>Grant</strong> to save the permissions.</li> 
  </ol> </li> 
</ol> 
<h3>Enable data export using the AWS CLI</h3> 
<p>Configure metadata export using the <code>PutDataExportConfiguration</code> API. The <a href="https://aws.amazon.com/datazone/" rel="noopener noreferrer" target="_blank">Amazon DataZone</a> service automatically creates an S3 table bucket named <code>aws-sagemaker-catalog</code> with an <code>asset_metadata</code> namespace, and schedules a daily export job.&nbsp;Asset metadata is exported once daily around midnight local time per AWS Region.</p> 
<p>The SageMaker Domain identifier is available on domain detail page in the <a href="https://aws.amazon.com/console/" rel="noopener noreferrer" target="_blank">AWS Management Console</a>. Accessing the asset table through the S3 Tables console or the Data tab in SageMaker Unified Studio can require up to 24 hours.</p> 
<p>AWS CLI command to enable SageMaker catalog export:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws datazone put-data-export-configuration --domain-identifier &lt;domain-id&gt; --region &lt;region&gt; --enable-export</code></pre> 
</div> 
<p>Use this AWS CLI command to validate the configuration is enabled:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">aws datazone get-data-export-configuration --domain-identifier &lt;domain-id&gt;&nbsp;--region &lt;region&gt;
{
&nbsp;&nbsp; &nbsp;"isExportEnabled": true,
&nbsp;&nbsp; &nbsp;"status": "COMPLETED",
&nbsp;&nbsp; &nbsp;"s3TableBucketArn": "arn:aws:s3tables:&lt;region&gt;:&lt;account-id&gt;:bucket/aws-sagemaker-catalog",
&nbsp;&nbsp; &nbsp;"createdAt": "2025-11-26T18:24:02.150000+00:00",
&nbsp;&nbsp; &nbsp;"updatedAt": "2026-02-23T19:33:40.987000+00:00"
}</code></pre> 
</div> 
<h3>Access the exported asset table</h3> 
<ol> 
 <li>Navigate to Amazon SageMaker <strong>Domains</strong> in the AWS Management Console.</li> 
 <li>Select your domain and select <strong>Open</strong>. <p></p>
  <div class="wp-caption alignnone" id="attachment_90412" style="width: 1440px;">
   <img alt="Amazon SageMaker Domains management page showing an Identity Center based domain with Available status, created February 26, 2026, with Open unified studio button highlighted" class="size-full wp-image-90412" height="313" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-5.jpeg" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-90412">Figure 5 – Open Amazon SageMaker Unified Studio</p>
  </div></li> 
 <li>In SageMaker Unified Studio, choose a project from the <strong>Select a project</strong> dropdown list.</li> 
 <li>To query SageMaker catalog data, select <strong>Build</strong> in the menu bar and then choose <strong>Query Editor</strong>. To create a new project, follow the instructions in the <a href="https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/getting-started-create-a-project.html" rel="noopener noreferrer" target="_blank">Amazon SageMaker Unified Studio User Guide</a>. <p></p>
  <div class="wp-caption alignnone" id="attachment_90411" style="width: 1439px;">
   <img alt="SageMaker Unified Studio project overview dashboard showing IDE and Applications, Data Analysis and Integration with Query Editor highlighted, Orchestration, and Machine Learning and Generative AI categories" class="size-full wp-image-90411" height="620" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-6.jpeg" width="1429" />
   <p class="wp-caption-text" id="caption-attachment-90411">Figure 6 – Open SageMaker Unified Studio Query Editor</p>
  </div></li> 
</ol> 
<p>The&nbsp;<code>asset_metadata.asset</code>&nbsp;table is available in Data explorer. Use <strong>Data explorer</strong> to view the schema and query data to perform analytics from.</p> 
<ol start="5"> 
 <li>Expand <strong>Catalogs</strong> in Data explorer. Then, select and expand <strong>s3tablecatalog, aws-sagemaker-catalog</strong>, <strong>asset_metadata,</strong> and <strong>asset.</strong></li> 
 <li>Test querying the catalog with <code>SELECT * FROM asset_metadata.asset LIMIT 10;</code>.</li> 
</ol> 
<div class="wp-caption alignleft" id="attachment_90410" style="width: 1439px;">
 <img alt="SageMaker Unified Studio Query Editor with Data Explorer showing Lakehouse hierarchy including s3tablescatalog, aws-sagemaker-catalog, asset_metadata database, and asset table schema with SQL SELECT query" class="wp-image-90410 size-full" height="731" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-7.jpeg" width="1429" />
 <p class="wp-caption-text" id="caption-attachment-90410">Figure 7 – Query SageMaker catalog</p>
</div> 
<h2>Queries for observability and analytics</h2> 
<p>With setup complete, execute queries to gain insights on catalog usage and changes. To monitor asset growth, and view how the data catalog has grown over the last five days:</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">SELECT 
 &nbsp;&nbsp; DATE (snapshot_time) as date,
 &nbsp;&nbsp; COUNT (*) as total_assets
FROM asset_metadata.asset
WHERE 
 &nbsp;&nbsp; &nbsp;DATE (snapshot_time) &gt;= CURRENT_DATE - INTERVAL '5' DAY
GROUP BY DATE (snapshot_time)
ORDER BY date DESC;</code></pre> 
</div> 
<div class="wp-caption alignleft" id="attachment_90409" style="width: 1439px;">
 <img alt="SageMaker Unified Studio Query Editor showing SQL aggregation query on asset_metadata.asset table with results displaying date and total_assets columns, returning 42 assets for March 7-8, 2026&quot;" class="wp-image-90409 size-full" height="730" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-8.jpeg" width="1429" />
 <p class="wp-caption-text" id="caption-attachment-90409">Figure 8 – Query asset growth</p>
</div> 
<p>Use the catalog to track metadata changes to determine which assets gained descriptions or ownership over time. Use this query to identify assets that gained business descriptions over the past five days by comparing today’s snapshot with the earlier snapshot.</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">SELECT
 &nbsp;&nbsp; t.asset_id,
 &nbsp;&nbsp; t.resource_name,
 &nbsp;&nbsp; p.business_description as description_before,
 &nbsp;&nbsp; t.business_description as description_now
FROM asset_metadata.asset t
JOIN asset_metadata.asset p ON t.asset_id = p.asset_id
WHERE DATE(t.snapshot_time) = CURRENT_DATE
 &nbsp;&nbsp; AND DATE(p.snapshot_time) = CURRENT_DATE - INTERVAL '5' DAY
 &nbsp;&nbsp; AND p.business_description IS NULL
 &nbsp;&nbsp; AND t.business_description IS NOT NULL;</code></pre> 
</div> 
<p>Investigate asset values at a specific point in time using this query to retrieve metadata from any snapshot date.</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">SELECT
 &nbsp;&nbsp; &nbsp;asset_id,
 &nbsp;&nbsp; &nbsp;resource_name,
 &nbsp;&nbsp; &nbsp;business_description,
 &nbsp;&nbsp; &nbsp;extended_metadata['owningEntityId'] as owner,
 &nbsp;&nbsp; &nbsp;snapshot_time
FROM asset_metadata.asset
WHERE asset_id = 'your-asset-id'
 &nbsp;&nbsp; &nbsp;AND DATE(snapshot_time) = DATE('2025-11-26');</code></pre> 
</div> 
<h2>Clean up resources</h2> 
<p>To avoid ongoing charges, clean up the resources created in this walkthrough:</p> 
<ol> 
 <li><strong>Disable metadata export:</strong></li> 
</ol> 
<p>Disable the daily metadata export to stop new snapshots:</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">aws datazone put-data-export-configuration \
  --domain-identifier &lt;domain-id. \
  --no-enable-export \
  --region &lt;region&gt;</code></pre> 
</div> 
<ol start="2"> 
 <li><strong>Delete S3 Tables resources:</strong></li> 
</ol> 
<p>Optionally, delete the S3 Tables namespace containing the exported metadata to remove historical snapshots and stop storage charges. For instructions on how to delete S3 tables, see <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-delete.html" rel="noopener noreferrer" target="_blank">Deleting an Amazon S3 table</a> in the Amazon Simple Storage Service User Guide.</p> 
<h2>Conclusion</h2> 
<p>In this post, you enabled the metadata export feature of SageMaker Catalog and used SQL queries to gain visibility into your asset inventory. The feature converts asset metadata into Apache Iceberg tables partitioned by snapshot date, so you can perform time-travel queries, monitor catalog growth, track metadata completeness, and audit historical asset states. This provides a repeatable, low-overhead way to maintain catalog health and meet governance requirements over time.</p> 
<p>To learn more about Amazon SageMaker Catalog, see the&nbsp;<a href="https://aws.amazon.com/sagemaker/catalog/" rel="noopener noreferrer" target="_blank">Amazon SageMaker Catalog documentation</a>. To explore Apache Iceberg table formats and time-travel queries, see the&nbsp;<a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html" rel="noopener noreferrer" target="_blank">Amazon S3 Tables documentation</a>.</p> 
<hr /> 
<h3>About the Authors</h3> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Photo of Author Ramesh Singh" class="alignleft size-full wp-image-90408" height="134" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-9.png" width="100" />
  </div> 
  <p><a href="http://www.linkedin.com/in/ramesh-harisaran-singh" rel="noopener noreferrer" target="_blank">Ramesh</a>&nbsp;is a Senior Product Manager Technical (External Services) at AWS in Seattle, Washington, currently with the Amazon SageMaker team. He is passionate about building high-performance ML/AI and analytics products that help enterprise customers achieve their critical goals using cutting-edge technology.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Photo of Author Pradeep Misra" class="alignleft size-full wp-image-90407" height="130" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-10.png" width="100" />
  </div> 
  <p><a href="https://www.linkedin.com/in/pradeep-m-326258a/" rel="noopener noreferrer" target="_blank">Pradeep</a>&nbsp;is a Principal Analytics and Applied AI Solutions Architect at AWS. He is passionate about solving customer challenges using data, analytics, and Applied AI. Outside of work, he likes exploring new places and playing badminton with his family. He also likes doing science experiments, building LEGOs, and watching anime with his daughters.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Photo of Author - Rohith Kayathi" class="alignleft size-full wp-image-90406" height="203" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-11.png" width="190" />
  </div> 
  <p><a href="https://www.linkedin.com/in/rohith-kayathi/" rel="noopener noreferrer" target="_blank">Rohith</a> is a Senior Software Engineer at Amazon Web Services (AWS) working with Amazon SageMaker team. He leads business data catalog, generative AI–powered metadata curation, and lineage solutions. He is passionate about building large-scale distributed systems, solving complex problems, and setting the bar for engineering excellence for his team.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Photo of AUthor - Steve Phillips" class="alignleft size-full wp-image-90405" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/BDB-5843-image-12.jpeg" width="120" />
  </div> 
  <p><a href="https://www.linkedin.com/in/stevephillipsca" rel="noopener noreferrer" target="_blank">Steve</a> is a Principal Technical Account Manager and Analytics specialist at AWS in the North America region. Steve currently focuses on data warehouse architectural design, data lakes, data ingestion pipelines, and cloud distributed architectures.</p> 
 </div> 
</footer>
