---
title: "A guide to capacity planning for Airflow worker pool in Amazon MWAA"
url: "https://aws.amazon.com/blogs/big-data/a-guide-to-capacity-planning-for-airflow-worker-pool-in-amazon-mwaa/"
date: "Fri, 01 May 2026 15:42:45 +0000"
author: "Boyko Radulov"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>In our previous post, <a href="https://aws.amazon.com/blogs/big-data/a-guide-to-airflow-worker-pool-optimization-in-amazon-mwaa/">A guide to Airflow worker pool optimization in Amazon MWAA</a>, we explored when adding workers to your Amazon Managed Workflows for Apache Airflow (Amazon MWAA) environment actually solves performance issues, and when it doesn’t. We walked through patterns like high CPU utilization and long queue times where scaling may be appropriate, and anti-patterns like misconfigured Airflow settings and memory leaks where adding workers only masks the real problem. The key takeaway was clear: optimize first, scale second, and always let data drive the decision.</p> 
<p>But what happens after you’ve done the optimization work? Your DAGs are efficient, your configurations are tuned, and your environment is running well. Then the business comes knocking: new regulatory requirements, additional data pipelines, expanded reporting. The workload is about to grow, and this time, you genuinely need more capacity.</p> 
<p>This is where capacity planning comes in. Knowing how many workers to provision, before the new workload hits production, is the difference between a smooth rollout and a 5 AM SLA breach. In this post, we walk through a practical capacity planning framework for Amazon MWAA worker pools. Using a real-world financial services scenario, we show how to assess your current capacity, project future needs, calculate the right number of base workers, and set up monitoring to keep your environment healthy as workloads evolve.</p> 
<p><strong>Scenario:</strong> A financial services company needs to plan capacity for a 25% directed acyclic graph (DAG) increase to support new regulatory reporting requirements.</p> 
<h2><strong>Current vs projected state</strong></h2> 
<p>The following table compares the current and expected state after adding 25% more DAGs.</p> 
<p>&nbsp;</p> 
<table border="1px" cellpadding="10px"> 
 <tbody> 
  <tr> 
   <td></td> 
   <td>Metric</td> 
   <td>Current</td> 
   <td>Projected</td> 
   <td>Change</td> 
  </tr> 
  <tr> 
   <td>1</td> 
   <td><strong>DAGs</strong></td> 
   <td>20</td> 
   <td>25</td> 
   <td>25%</td> 
  </tr> 
  <tr> 
   <td>2</td> 
   <td><strong>Peak Tasks (5-7 AM)</strong></td> 
   <td>80</td> 
   <td>104</td> 
   <td>+24 tasks</td> 
  </tr> 
  <tr> 
   <td>3</td> 
   <td><strong>Environment Class</strong></td> 
   <td>mw1.medium</td> 
   <td>mw1.medium</td> 
   <td>No change</td> 
  </tr> 
  <tr> 
   <td>4</td> 
   <td><strong>Base Workers</strong></td> 
   <td>8</td> 
   <td>11</td> 
   <td>+3 workers</td> 
  </tr> 
  <tr> 
   <td>5</td> 
   <td><strong>Tasks per Worker</strong></td> 
   <td>10 (mw1.medium default)</td> 
   <td>10</td> 
   <td>No change</td> 
  </tr> 
  <tr> 
   <td>6</td> 
   <td><strong>Available Capacity</strong></td> 
   <td>80 slots (8 × 10)</td> 
   <td>110 slots (11 × 10)</td> 
   <td>+30 slots</td> 
  </tr> 
  <tr> 
   <td>7</td> 
   <td><strong>Peak Utilization</strong></td> 
   <td>100% (80/80 slots) <img alt="⚠" class="wp-smiley" src="https://s.w.org/images/core/emoji/14.0.0/72x72/26a0.png" style="height: 1em;" /></td> 
   <td>95% (104/110 slots)</td> 
   <td>Improved</td> 
  </tr> 
  <tr> 
   <td>8</td> 
   <td><strong>Critical SLA</strong></td> 
   <td>7 AM market open</td> 
   <td>7 AM market open</td> 
   <td>No tolerance</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Capacity planning goal:</strong> Reduce utilization from 100% to 95% to maintain service level agreement (SLA) compliance and handle unexpected spikes.</p> 
<p><strong>Understanding current capacity:</strong> The environment currently runs 8 base workers, providing 80 concurrent task slots (8 workers × 10 tasks per worker). During the 5-7 AM peak with 80 concurrent tasks, this represents 100% utilization, a risky level that leaves no headroom for unexpected spikes or volatility.<br /> With the planned addition of 5 new regulatory reporting DAGs, peak concurrent tasks will grow to 104. To maintain healthy operations with adequate buffer, we need to increase to 11 base workers (110 slots), resulting in 95% peak utilization with 6 slots of breathing room.</p> 
<p><strong>Why 100% utilization is risky: </strong>Running at 100% task utilization means:</p> 
<ul> 
 <li>Zero buffer for unexpected spikes</li> 
 <li>Any additional task causes immediate queuing</li> 
 <li>No room for market volatility or data volume increases</li> 
 <li>High risk of SLA breaches during unpredictable events</li> 
</ul> 
<p><strong>Best practice: Maintain at least 5-15% headroom (85-95% utilization) for production workloads with critical SLAs.</strong></p> 
<p><strong>Why this sizing:</strong></p> 
<ul> 
 <li><strong>Current:</strong> 80 tasks ÷ 80 slots = 100% utilization (at capacity – risky!)</li> 
 <li><strong>Projected:</strong> 104 tasks ÷ 110 slots = 95% utilization (healthy with buffer)</li> 
 <li><strong>Buffer:</strong> 6 slots (5% headroom) protects against unexpected volatility spikes</li> 
 <li><strong>SLA protection:</strong> Adequate headroom prevents queuing during normal operations</li> 
</ul> 
<h2><strong>Capacity analysis</strong></h2> 
<p>Every team asks the same critical question: <strong>“How many workers do I need</strong>?” The process is to identify your peak concurrent tasks from <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/accessing-metrics-cw-container-queue-db.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch metrics,</a> dividing by your environment’s tasks-per-worker capacity, and adding a 5%-15% safety buffer.</p> 
<h3><strong>Step 1: Identifying peak concurrent tasks from Amazon CloudWatch</strong></h3> 
<p>To determine your peak workload, you need to analyze RunningTasks and QueuedTasks CloudWatch metrics for your Amazon MWAA environment. Navigate to Amazon CloudWatch and query the following key metrics:</p> 
<h4><strong>Primary metrics for capacity planning:</strong></h4> 
<ul> 
 <li><strong>RunningTasks:</strong> Number of tasks currently executing across all workers. This shows your actual concurrent task load.</li> 
 <li><strong>QueuedTasks:</strong> Number of tasks waiting for available worker slots. High values indicate insufficient capacity.</li> 
 <li><strong>AvailableWorkers:</strong> Current number of active workers in your environment.</li> 
</ul> 
<p><strong>How to find peak concurrent tasks:</strong></p> 
<ol> 
 <li>Open the Amazon CloudWatch Console. 
  <ul> 
   <li>Choose <strong>Metrics</strong>.</li> 
   <li>Choose the <strong>MWAA </strong>namespace.</li> 
  </ul> </li> 
 <li>Select your environment name.</li> 
 <li>Add the <code>RunningTasks</code> metric.</li> 
 <li>Set time range to last 7-30 days.</li> 
 <li>Change statistic to <strong>Maximum</strong>.</li> 
 <li>Identify the highest value during your peak hours (for example, 5-7 AM).</li> 
</ol> 
<p><strong>Example query:</strong><br /> Note: The following query is conceptual and does not directly translate to Amazon CloudWatch-specific language. Please refer to the <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/query_with_cloudwatch-metrics-insights.html" rel="noopener noreferrer" target="_blank">Query your CloudWatch metrics with CloudWatch Metrics Insights</a> for more information.</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">SELECT MAX(RunningTasks) AS PeakConcurrentTasks
FROM MWAA_Metrics
WHERE Environment = 'prod-airflow'
  AND timestamp BETWEEN '2024-10-01' AND '2024-10-31'
  AND HOUR(timestamp) BETWEEN 5 AND 7;</code></pre> 
</div> 
<p>In our scenario, this analysis revealed <strong>80 concurrent tasks</strong> during the 5-7 AM window. With the planned 25% DAG increase, we project this will grow to <strong>104 concurrent tasks</strong>.</p> 
<h3>Step 2: Calculate required workers</h3> 
<p>To calculate the number of required workers without queuing any tasks, use the following formula: <strong>Peak concurrent tasks ÷ Tasks per worker × Safety buffer = Required workers</strong></p> 
<p>In the projected scenario with 104 tasks at peak hours, using mw1.medium environment with default concurrency configuration and having a 5% safety buffer, we need 11 workers</p> 
<ul> 
 <li>104 peak tasks ÷ 10 tasks per worker × 1.06 buffer = 11 workers required to handle your workload without queuing during busiest periods.</li> 
</ul> 
<h2>Capacity monitoring and triggers</h2> 
<p>There are a few important Amazon CloudWatch metrics to monitor for environment health.</p> 
<h3>Key metrics to monitor</h3> 
<p>Monitor these five critical Amazon CloudWatch metrics to detect capacity issues:</p> 
<ul> 
 <li>QueuedTasks (&gt;10 for &gt;5 minutes indicates insufficient capacity)</li> 
 <li>RunningTasks (consistently at maximum suggests the need for more workers)</li> 
 <li>AdditionalWorkers (active for more than 6 hours daily signals the permanent worker problem)</li> 
 <li>Worker CPU (&gt;85% sustained requires environment class upgrade or workload optimization)</li> 
 <li>Task Duration (+15% increase means reduced effective capacity per worker).</li> 
</ul> 
<p>These metrics provide early warning signals to adjust capacity before SLA breaches occur.</p> 
<p>&nbsp;</p> 
<table border="1px" cellpadding="10px"> 
 <tbody> 
  <tr> 
   <td></td> 
   <td>Metric</td> 
   <td>Threshold</td> 
   <td>Action</td> 
  </tr> 
  <tr> 
   <td>1</td> 
   <td><strong>QueuedTasks</strong></td> 
   <td>&gt;10 for &gt;5 minutes</td> 
   <td>Investigate capacity</td> 
  </tr> 
  <tr> 
   <td>2</td> 
   <td><strong>RunningTasks</strong></td> 
   <td>Consistently at max</td> 
   <td>Increase base workers</td> 
  </tr> 
  <tr> 
   <td>3</td> 
   <td><strong>AdditionalWorkers</strong></td> 
   <td>Active &gt;6 hours daily</td> 
   <td>Increase base workers</td> 
  </tr> 
  <tr> 
   <td>4</td> 
   <td><strong>Worker CPU</strong></td> 
   <td>&gt;85% sustained</td> 
   <td>Upgrade environment class</td> 
  </tr> 
  <tr> 
   <td>5</td> 
   <td><strong>Task Duration</strong></td> 
   <td>+15% increase</td> 
   <td>Review capacity per worker</td> 
  </tr> 
 </tbody> 
</table> 
<h3>Amazon CloudWatch monitoring queries</h3> 
<p>Note: The following queries are conceptual and do not directly translate to Amazon CloudWatch-specific language. Please refer to the <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/query_with_cloudwatch-metrics-insights.html" rel="noopener noreferrer" target="_blank">Query your CloudWatch metrics with CloudWatch Metrics Insights</a> for more information.</p> 
<ul> 
 <li>Queue depth during peak hours 
  <div class="hide-language"> 
   <pre><code class="lang-sql">SELECT AVG(QueuedTasks)
FROM MWAA_Metrics
WHERE Environment = 'prod-airflow'
  AND timestamp BETWEEN '05:00' AND '07:00'
GROUP BY 5m;</code></pre> 
  </div> </li> 
 <li>Worker utilization efficiency 
  <div class="hide-language"> 
   <pre><code class="lang-sql">SELECT AVG(RunningTasks) / AVG(AvailableWorkers * 5) * 100 AS UtilizationPercent
FROM MWAA_Metrics
WHERE Environment = 'prod-airflow';</code></pre> 
  </div> </li> 
 <li>Detect permanent worker problem 
  <div class="hide-language"> 
   <pre><code class="lang-sql">SELECT DATE(timestamp) AS date,
       AVG(AdditionalWorkers) AS avg_additional,
       MAX(AdditionalWorkers) AS max_additional
FROM MWAA_Metrics
WHERE AdditionalWorkers &gt; 0
GROUP BY DATE(timestamp)
HAVING AVG(AdditionalWorkers) &gt; 5;</code></pre> 
  </div> </li> 
</ul> 
<h3><strong>Setting up alerts</strong></h3> 
<p>You can configure these alarms to identify problems as soon as they are introduced.</p> 
<h4><strong>Recommended Amazon CloudWatch alarms:</strong></h4> 
<ol> 
 <li><strong>High queue depth alert</strong> 
  <ul> 
   <li>Metric: QueuedTasks</li> 
   <li>Threshold: &gt; 10 for 2 consecutive 5-minute periods</li> 
   <li>Action: Notify operations team</li> 
  </ul> </li> 
 <li><strong>Permanent worker detection</strong> 
  <ul> 
   <li>Metric: AdditionalWorkers</li> 
   <li>Threshold: &gt; 0 for 6+ hours</li> 
   <li>Action: Review capacity planning</li> 
  </ul> </li> 
 <li><strong>SLA risk alert</strong> 
  <ul> 
   <li>Metric: QueuedTasks during 5-7 AM window</li> 
   <li>Threshold: &gt; 5 tasks</li> 
   <li>Action: Page on-call engineer</li> 
  </ul> </li> 
</ol> 
<h3><strong>When to revisit capacity planning</strong></h3> 
<p>Conduct quarterly scheduled reviews to analyze trends and project growth. Also run immediate trigger-based assessments when:</p> 
<ul> 
 <li>DAG count increases &gt;10% (or more than your safety buffer)</li> 
 <li>Performance degrades</li> 
 <li>Cost anomalies appear (indicating permanent workers)</li> 
 <li>Any SLA breach occurs.</li> 
</ul> 
<p>This dual approach provides proactive capacity management while enabling rapid response to emerging issues.</p> 
<p>&nbsp;</p> 
<table border="1px" cellpadding="10px"> 
 <tbody> 
  <tr> 
   <td></td> 
   <td>Trigger</td> 
   <td>Frequency</td> 
   <td>Action</td> 
  </tr> 
  <tr> 
   <td>1</td> 
   <td><strong>Scheduled Review</strong></td> 
   <td>Quarterly</td> 
   <td>Analyze trends, project growth</td> 
  </tr> 
  <tr> 
   <td>2</td> 
   <td><strong>DAG Growth</strong></td> 
   <td>&gt;10% increase</td> 
   <td>Recalculate capacity needs</td> 
  </tr> 
  <tr> 
   <td>3</td> 
   <td><strong>Performance Degradation</strong></td> 
   <td>As observed</td> 
   <td>Immediate capacity assessment</td> 
  </tr> 
  <tr> 
   <td>4</td> 
   <td><strong>Cost Anomalies</strong></td> 
   <td>Monthly</td> 
   <td>Check for permanent workers</td> 
  </tr> 
  <tr> 
   <td>5</td> 
   <td><strong>SLA Breaches</strong></td> 
   <td>Any occurrence</td> 
   <td>Emergency capacity review</td> 
  </tr> 
 </tbody> 
</table> 
<h2><strong>Decision matrix</strong></h2> 
<p>The framework presents three capacity planning approaches, each optimized for different organizational priorities.</p> 
<p>The <strong>Full Base Worker Provisioning strategy</strong> (the conservative path) sets base workers equal to the calculated requirement, eliminating queue times during peak periods and guaranteeing SLA compliance with predictable fixed costs, while automatic scaling handles only unexpected spikes—ideal for mission-critical workloads with strict SLA requirements.</p> 
<p>The <strong>Minimal Base + Automatic Scaling approach</strong> (the cost-focused path) maintains minimal base workers at current levels and relies heavily on automatic scaling, accepting 3-5 minute delays during peak periods and SLA breach risks in exchange for lower baseline costs, though this requires intensive monitoring and carries explicit warnings about high SLA risk.</p> 
<p>The <strong>Hybrid Approach</strong> (the balanced path) provisions base workers at 80% of the calculated requirement with automatic scaling covering the remaining 20%, resulting in 2-3 minute delays during spikes while balancing cost against performance—suitable for moderate SLA requirements with some budget constraints.</p> 
<p>The comparison table contrasts queue times (under 30 seconds versus 2-3 minutes versus 3-5 minutes), SLA compliance levels (guaranteed versus high probability versus at-risk during peak), and ideal use cases (mission-critical predictable workloads versus moderate SLA requirements with budget constraints versus development environments with flexible SLA tolerance), enabling teams to make informed provisioning decisions aligned with their operational requirements and financial constraints.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-1.jpeg" /></p> 
<h2><strong>Key takeaway</strong></h2> 
<p>Effective capacity planning prevents both under-provisioning (SLA breaches) and over-provisioning (cost overruns).</p> 
<h3><strong>Capacity planning principles</strong></h3> 
<ol> 
 <li><strong>Calculate capacity needs BEFORE adding workload</strong> – Use peak task projections with 5-15% safety buffer</li> 
 <li><strong>Size minimum workers for peak demand</strong> – Don’t rely on automatic scaling for predictable loads</li> 
 <li><strong>Use automatic scaling only for unexpected spikes</strong> – Treat as safety net, not primary capacity</li> 
 <li><strong>Target 85-95% utilization during peak hours</strong> – Ensures headroom for unexpected growth</li> 
 <li><strong>Plan 5-15% headroom for unexpected growth</strong> – Production often differs from testing</li> 
 <li><strong>Monitor AdditionalWorkers metric</strong> – If active &gt;6 hours daily, increase base workers</li> 
 <li><strong>Review quarterly + trigger-based assessments</strong> – Regular reviews plus immediate action on issues</li> 
 <li><strong>Balance cost and performance based on SLA criticality</strong> – Business impact justifies infrastructure investment</li> 
</ol> 
<h3><strong>Success metrics</strong></h3> 
<ul> 
 <li><strong>Queue efficiency:</strong> Average queue time &lt;30 seconds during peak</li> 
 <li><strong>SLA compliance:</strong> &gt;99.5% of critical tasks complete on time</li> 
 <li><strong>Resource utilization:</strong> 85-95% during peak hours (optimal efficiency)</li> 
 <li><strong>Cost predictability:</strong> &lt;10% variance in monthly worker costs</li> 
</ul> 
<h2><strong>Conclusion</strong></h2> 
<p>Capacity planning is not a one-time exercise. It’s an ongoing discipline. The framework we’ve outlined gives you a repeatable process: measure your current peak utilization through CloudWatch metrics, project growth based on incoming workloads, calculate the required workers with an appropriate safety buffer, and monitor continuously to catch drift before it becomes an outage.</p> 
<p>The financial services scenario in this post illustrates a common reality: running at 100% utilization during peak hours leaves zero room for the unexpected. By sizing to 95% peak utilization with a modest buffer, the team gained the headroom needed to absorb volatility without risking their 7 AM market-open SLA.</p> 
<p>Whether you choose full base worker provisioning for mission-critical pipelines, a hybrid approach for moderate SLA requirements, or lean on automatic scaling for development workloads, the right strategy depends on your business context, not a one-size-fits-all rule. Pair your capacity plan with the CloudWatch alarms and review triggers we covered, and you’ll catch capacity gaps early.</p> 
<p>Combined with the optimization-first approach from Part 1, you now have a complete toolkit: diagnose before you scale, optimize before you provision, and plan before you deploy. Your MWAA environment and your on-call engineers will thank you.</p> 
<p>To get started, visit the <a href="https://aws.amazon.com/managed-workflows-for-apache-airflow/" rel="noopener noreferrer" target="_blank">Amazon MWAA product page</a> and the <a href="https://console.aws.amazon.com/mwaa/home" rel="noopener noreferrer" target="_blank">Amazon MWAA console page</a>.</p> 
<p>If you have questions or want to share your MWAA capacity planning, leave a comment.</p> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Boyko Radulov" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-2.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Boyko Radulov</h3> 
  <p>Boyko is a Senior Cloud Support Engineer at Amazon Web Services (AWS), Amazon MWAA and AWS Glue Subject Matter Expert. He works closely with customers to build and optimize their workloads on AWS while reducing the overall cost. Beyond work, he is passionate about sports and travelling.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Kamen Sharlandjiev" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-3.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Kamen Sharlandjiev</h3> 
  <p>Kamen is a Principal Big Data and ETL Solutions Architect, Amazon MWAA and AWS Glue ETL expert. He’s on a mission to make life easier for customers who are facing complex data integration and orchestration challenges. His secret weapon? Fully managed AWS services that can get the job done with minimal effort. Follow Kamen on <a href="https://www.linkedin.com/in/ksharlandjiev/" rel="noopener noreferrer" target="_blank"><em>LinkedIn</em></a> to keep up to date with the latest Amazon MWAA and AWS Glue features and news.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Venu Thangalapally" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-4.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Venu Thangalapally</h3> 
  <p>Venu is a Senior Solutions Architect at AWS, based in Chicago, with deep expertise in cloud architecture, data and analytics, containers, and application modernization. He partners with financial service industry customers to translate business goals into secure, scalable, and compliant cloud solutions that deliver measurable value. Venu is passionate about using technology to drive innovation and operational excellence.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Harshawardhan Kulkarni" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-5.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Harshawardhan Kulkarni</h3> 
  <p>Harshawardhan is a Partner Technical Account Manager at AWS, Amazon MWAA Subject Matter Expert. Based in Dublin Ireland, he partners with Enterprise Customers across EMEA to help navigate complex workflows and orchestration challenges while ensuring best practice implementation. Outside of work, he enjoys traveling and spending time with his family.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Andrew McKenzie" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/29/BDB-4941-6.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Andrew McKenzie</h3> 
  <p>Andrew is a Data Engineer and Educator who uses deep technical expertise from his time at AWS. As a former Amazon MWAA Subject Matter Expert, he now focuses on building data solutions and teaching data engineering best practices.</p> 
 </div> 
</footer>
