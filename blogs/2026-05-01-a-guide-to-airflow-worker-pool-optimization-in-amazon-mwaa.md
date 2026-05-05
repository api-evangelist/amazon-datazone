---
title: "A guide to Airflow worker pool optimization in Amazon MWAA"
url: "https://aws.amazon.com/blogs/big-data/a-guide-to-airflow-worker-pool-optimization-in-amazon-mwaa/"
date: "Fri, 01 May 2026 15:41:26 +0000"
author: "Boyko Radulov"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Optimizing the Airflow worker pool configuration in <a href="http://aws.amazon.com/managed-workflows-for-apache-airflow" rel="noopener noreferrer" target="_blank">Amazon Managed Workflows for Apache Airflow</a> (Amazon MWAA), the AWS fully managed Apache Airflow service, is an important yet often overlooked strategy for scaling workflow operations. Tasks queued for longer periods can create the illusion that additional workers are the solution, when in reality the root cause might lie elsewhere. The decision to scale isn’t always straightforward. DevOps engineers and system administrators frequently face the challenge of determining whether adding more workers will solve their performance issues or only increase operational cost without addressing the root cause.</p> 
<p>This post explores different patterns for worker scaling decisions in Amazon MWAA, focusing on the task pool mechanism and its relationship to worker allocation. By examining specific scenarios and providing a practical decision framework, this post helps you determine whether adding workers is the right solution for your performance challenges, and if so, how to implement this scaling effectively.</p> 
<h1>Main patterns</h1> 
<p>This section discusses the most frequently seen problems that raise the question if adding additional workers would improve the health of your environment.</p> 
<h2>High CPU</h2> 
<p>Airflow serves as a workflow management platform that coordinates and schedules tasks to be run on external processing services. It acts as a central orchestrator that can trigger and monitor tasks across various data processing systems like <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a>, <a href="https://aws.amazon.com/batch/" rel="noopener noreferrer" target="_blank">AWS Batch</a>, <a href="https://aws.amazon.com/emr/" rel="noopener noreferrer" target="_blank">Amazon EMR</a>, and other specialized data processing tools. Rather than processing data itself, Airflow’s strength lies in managing complex workflows and coordinating jobs between different systems and services.</p> 
<p>In Analytics and Big Data environments, there is a prevalent misconception that saturated resources automatically warrant adding more capacity. However, for Amazon MWAA, understanding your workflow characteristics and optimization opportunities should precede scaling decisions.</p> 
<p>As you scale up your workflows, resource utilization of the Airflow clusters naturally increases. When workers consistently operate at full capacity, it may seem intuitive to add additional compute resources. However, this approach often masks underlying inefficiencies rather than resolving them.</p> 
<p>For example, in Amazon MWAA if you are running a single task that is consuming 100% of the available CPU on your Amazon MWAA worker, adding additional workers will not resolve the problem as the task is not optimized nor split into smaller parts. As such, increasing the number of minimum workers will not bring the expected effect but will only increase the operating costs.</p> 
<p>When your Amazon MWAA workers are consistently running above 90% CPU or Memory utilization, you’ve reached a critical decision point. Before taking actions, it is essential to understand the root cause. You have three primary options:</p> 
<ol> 
 <li>Scale horizontally by adding additional workers to distribute the load.</li> 
 <li>Scale vertically by upgrading to a larger environment class for more resources per worker.</li> 
 <li>Optimize your DAGs and scheduling patterns to be more efficient and consume fewer resources.</li> 
</ol> 
<p>Each approach addresses different underlying issues, and choosing the right path depends on identifying whether you are facing a capacity constraint, resource-intensive task design, or workflow inefficiency. For guidance on optimization strategies, please refer to <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/best-practices-tuning.html" rel="noopener noreferrer" target="_blank">Performance tuning for Apache Airflow on Amazon MWAA</a>.</p> 
<p>To monitor the <code>CPUUtilization</code> and <code>MemoryUtilization</code> on the workers, refer to the <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/accessing-metrics-cw-container-queue-db.html#accessing-metrics-cw-container-queue-db-console" rel="noopener noreferrer" target="_blank">Accessing metrics in the Amazon CloudWatch console</a> and choose the corresponding metrics.</p> 
<ol> 
 <li>Select a time window long enough to show usage patterns.</li> 
 <li>Set period to <strong>1 Minute</strong>.</li> 
 <li>Set statistics to <strong>Maximum</strong>.</li> 
</ol> 
<h2>Long queue time</h2> 
<p>Sometimes Airflow tasks are stuck in a queued state for a long time, which prevents DAGs from completing on time.</p> 
<p>In Amazon MWAA, each environment class comes with configured minimum and maximum worker nodes. Each worker provides a pre-configured concurrency, which is the number of tasks that can run simultaneously on each worker at any given time. The behavior is controlled through <code>celery.worker_autoscale=(max,min).</code></p> 
<p>For example, if you have minimum 4 mw1.small workers, with default Airflow configuration, you will be able to run 20 concurrent tasks (4 workers x 5 max_tasks_per_worker). If your system suddenly requires more than 20 tasks to execute concurrently, this will result in an autoscaling event. Amazon MWAA will decide how to scale your workers efficiently, and trigger the process. The autoscaling process, however, requires additional time to provision new workers resulting in additional tasks in queued status. To mitigate this queuing issue, consider the following:</p> 
<ol> 
 <li>If the CPU utilization on the workers is low, increasing the <code>max</code> value in <code>celery.worker_autoscale=(max,min)</code> can reduce the time tasks stay in queued state as each worker will be able to process more tasks concurrently. Airflow worker can take tasks up to the defined task concurrency regardless of the availability of its own system resources. As a result, the base worker may reach 100% CPU/Memory utilization before Autoscaling takes effect.</li> 
 <li>If you do not want to increase the task concurrency on the workers, increasing the minimum worker count can also be beneficial because having more available workers allows a higher number of tasks to run concurrently.</li> 
</ol> 
<h2>Scheduling delays</h2> 
<p>Adding new DAGs can not only affect your system resources, but it can also create uneven scheduling patterns. Some DAGs may experience delayed execution because of resource competition, even when the overall environment metrics appear healthy. This scheduling skew often manifests as inconsistent task pickup times, where certain workflows consistently wait longer in the queue while others execute promptly.</p> 
<p>When <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/accessing-metrics-cw-container-queue-db.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch metrics</a> show increasing variance in task scheduling times, particularly during periods of high DAG activity, it signals the need for environment optimization. This scenario requires careful analysis of execution patterns and resource utilization to determine if:</p> 
<ol> 
 <li>While adding workers can help distribute the workload, this solution is most effective when the high utilization is primarily because of task execution load rather than DAG parsing or scheduling overhead. Adding more minimum workers will allow you to execute more tasks in parallel. For example, if you observe the value of <code>AWS/MWAA/ApproximateAgeOfOldestTask </code>to be steadily increasing, it means that the workers are not able to consume the messages from the queue fast enough. Additionally, you can also monitor the <code>AWS/MWAA/QueuedTasks</code> to identify similar patterns.</li> 
 <li>Upgrading the environment class would provide better scheduling capacity. If the Scheduler is showing signs of strain or if you’re seeing high resource utilization across all components, upgrading to a larger environment class might be the most appropriate solution. This provides more resources to both the Scheduler and Workers, allowing for better handling of increased DAG complexity and volume. To validate the same, use <code>AWS/MWAA/CPUUtilization</code> and <code>AWS/MWAA/MemoryUtilization</code> in the Cluster metrics and choose <code>Scheduler,</code> <code>BaseWorker</code> and <code>AdditionalWorker</code> metrics.</li> 
 <li>Restructuring DAG schedules would reduce resource contention.</li> 
</ol> 
<p>The key is to understand your workflow patterns and identify whether the scheduling delays are because of insufficient worker capacity or other environmental constraints.</p> 
<h1>Anti patterns</h1> 
<p>This section showcases the most common anti patterns which make MWAA users think that adding more workers will improve performance.</p> 
<h2>Underutilized workers</h2> 
<p>When evaluating Amazon MWAA performance bottlenecks, it’s important to distinguish resource constraints and DAG design inefficiencies before scaling the environment.</p> 
<p>Sometimes the Amazon MWAA environment has the capacity to run 100 tasks concurrently but your queue metrics (<code>AWS/MWAA/RunningTasks</code>) show only 20 tasks active most of the time with no tasks remaining in queued state. In such scenarios, you are advised to check <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/accessing-metrics-cw-container-queue-db.html#accessing-metrics-cw-container-queue-db-list" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> for consistently low CPU and memory usage on existing workers during peak workload times. If this is confirmed, it is usually an indication of inefficiencies in DAG design, scheduling patterns, or Airflow configuration.</p> 
<p>You have two primary options to address this:</p> 
<p>1. <strong>Downsize</strong>: If you do not expect your workload to increase, it is safe to assume you have over-provisioned your cluster. Start by removing any extra workers first and finally resolve to downsizing your environment class.</p> 
<p>2. <strong>Optimize</strong>: Fine tune your DAG scheduling and airflow configuration through Pools and Airflow configuration for concurrency to increase the throughput of your system.</p> 
<h2>Misconfigured Airflow configurations that create artificial bottlenecks</h2> 
<p>In Apache Airflow, performance bottlenecks often occur because of configuration settings, not actual resource constraints. At such times, DAG executions get delayed not because of insufficient compute, but because of incorrect concurrency configuration.</p> 
<p>Efficient use of Amazon MWAA requires reviewing not only resource utilization for Workers and Schedulers but also concurrency configurations for artificially created bottlenecks. Sometimes one restrictive configuration prevents the scaling benefits of larger environment or additional workers. Always audit Airflow configurations if performance seems limited even when system metrics suggest spare capacity.</p> 
<p><em><strong>Important consideration</strong>: Amazon Managed Workflows for Apache Airflow (Amazon MWAA) does not automatically update the worker concurrency configuration when you change the environment class. This behavior is important to understand when scaling your environment. If you initially create an mw1.small environment, where each worker can handle up to 5 concurrent tasks by default. When you upgrade to a medium environment class (which supports 10 concurrent tasks per worker by default), the concurrency setting <strong>remains at 5</strong> for in-place updated environments. You must manually update the concurrency configuration to take full advantage of the increased capacity available in the medium environment class.</em></p> 
<p>Because of this you need to also update the Airflow configurations that control concurrency whenever you update the environment class. To update the concurrency setting after upgrading your environment class, modify the <code>celery.worker_autoscale</code> configuration in your Apache Airflow configuration options. This makes sure your workers can process the maximum number of concurrent tasks supported by your new environment class.</p> 
<p>Other times, an Amazon MWAA environment can be constrained by <code>max_active_runs</code> or DAG concurrency controls instead of actual resource limits. These configuration-based throttles prevent tasks from running, even when the worker instances have available compute to handle the workload.</p> 
<p>There is an important distinction between the two. Configuration limits act as artificial caps on parallelism, while true resource limits indicate that workers are fully utilizing their CPU or memory capacity. Understanding which type of constraint affects your environment helps you determine whether to adjust configuration settings or scale your infrastructure.</p> 
<p>Adjusting Airflow configurations such as Pools, concurrency, max_active_runs solves performance problems without scaling workers. Some of the configurations you can use to control this behavior:</p> 
<ol> 
 <li><strong>max_active_runs_per_dag</strong> (DAG level): Controls how many DAG runs for a given DAG are allowed at the same time. If set to 2, only 2 DAG runs can run concurrently, even if there is plenty of worker capacity left. Extra runs queue, making the DAG executions slow even though workers are idle.</li> 
 <li><strong>max_active_tasks:</strong>Controls the concurrency field in a DAG definition (or setting at environment level) limits the number of tasks from the DAG running at any moment, regardless of overall system capacity or number of workers.</li> 
 <li><strong>Pools:</strong>Pools restrict how many tasks of a certain type (often resource heavy) can run at once. A pool with only 3 slots will throttle any tasks above 3 assigned to that pool, leaving workers idle.</li> 
 <li><strong>Execution timeouts and retries:</strong> If not tuned, failed tasks might fill up slots unnecessarily, stuck tasks can block worker slots and slow queue processing.</li> 
 <li><strong>Scheduling intervals and dependencies:</strong> Overlapping or inefficient scheduling may cause idle periods or excess contention for resources, affecting real throughput.</li> 
</ol> 
<p><strong>How Airflow configurations can override each other</strong></p> 
<p>Airflow has multiple layers of concurrency and scheduling controls. Some at the environment level, some at the DAG/task level, and others for pools. Sometimes more restrictive settings override more permissive ones, resulting in unexpected queue buildup.</p> 
<p><strong>DAG level vs Environment level:</strong> If “max_active_runs_per_dag” (DAG level) is lower than the environment-level “max_active_runs_per_dag” or system wide concurrency, the DAG setting is used, throttling tasks even if the environment could do more.</p> 
<p><strong>Task level overrides:</strong> Individual task definitions can have their own parameters like “max_active_tis_per_dag” which can cap runs per task and create a bottleneck if set lower than global settings.</p> 
<p><strong>Order of precedence:</strong> The most restrictive relevant configuration at any level (Environment, DAG, Task) effectively sets the upper bound for parallel task execution.</p> 
<table border="1px" cellpadding="10px"> 
 <tbody> 
  <tr> 
   <td><strong>Setting Location</strong></td> 
   <td><strong>Setting</strong></td> 
   <td><strong>Effect on task throughput</strong></td> 
  </tr> 
  <tr> 
   <td>Environment Level</td> 
   <td>parallelism</td> 
   <td>Max total tasks running on Scheduler</td> 
  </tr> 
  <tr> 
   <td>DAG Level</td> 
   <td>max_active_runs</td> 
   <td>Max simultaneous DAG runs</td> 
  </tr> 
  <tr> 
   <td>Task Level</td> 
   <td>concurrency</td> 
   <td>Max concurrent task for that DAG</td> 
  </tr> 
 </tbody> 
</table> 
<p>Performance issues often resemble resource exhaustion, but actually derive from overly restrictive configurations. Audit all the preceding parameters carefully. You can loosen restrictive values step by step and monitor their effect before deciding to scale your cluster further. This approach ensures optimal and cost-efficient usage of your cloud resources without paying for idle capacity.</p> 
<h2>Slow resource depletion from memory leaks</h2> 
<p>A common scenario for memory leak or slow resource depletion in Amazon MWAA is when DAGs and tasks begin to fail or slow down over time. Scaling workers or increasing environment size does not resolve the underlying issue. This happens because the root cause is not a lack of capacity but rather an application-level leak that causes persistent exhaustion.</p> 
<p>For example, as Airflow continuously runs tasks and parses DAGs over time, memory consumption can steadily increase across the environment. This might manifest as an Amazon MWAA metadata database experiencing declining FreeableMemory metrics despite consistent or even reduced workloads. When this occurs, database query performance gradually declines as memory resources become constrained for scheduler/worker &amp; metadata database, ultimately affecting overall environment responsiveness since Airflow depends heavily on its metadata database for critical operations. This scenario is similar to how an application might create database connections without properly closing them, leading to resource exhaustion over time.</p> 
<h3>Graph: Declining FreeableMemory and MemoryUtilization</h3> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/30/graph-freeablememory-memoryutilization-2026-04-30.png" /></p> 
<p><strong>Common causes:</strong></p> 
<ol> 
 <li><strong>Connection pool exhaustion:</strong> DAGs that fail to properly close database connections can lead to connection pool exhaustion and memory leaks in the database.</li> 
 <li><strong>Resource-intensive operations:</strong> Complex, long-running queries or XCOM operations against the metadata database can consume excessive memory.</li> 
 <li><strong>Inefficient DAG design:</strong> DAGs with numerous top-level Python calls can trigger database queries during DAG parsing. For instance, using variable.get() calls at the DAG level rather than at the task level creates unnecessary database load.</li> 
</ol> 
<p><strong>Recommended solutions:</strong></p> 
<ol> 
 <li><strong>Implement Amazon CloudWatch monitoring:</strong> Establish Amazon CloudWatch alarms for FreeableMemory with appropriate thresholds to detect issues early.</li> 
 <li><strong>Regular database maintenance:</strong> Perform scheduled database clean-up operations to purge historical data that is no longer needed.</li> 
 <li><strong>Optimize DAG code:</strong> Refactor DAGs to move database operations like variable.get() from the DAG level to the task level to reduce parsing overhead.</li> 
 <li><strong>Connection management:</strong> Make sure all database connections are properly closed after use to prevent connection pool exhaustion.</li> 
</ol> 
<p>By following the preceding recommendations you can maintain healthy memory utilization for the metadata database and maintain optimal performance of your Amazon MWAA environment without needing to scale workers.</p> 
<h1>Conclusion</h1> 
<p>The decision to add workers in Amazon MWAA environments requires careful consideration of multiple factors beyond simple task queue metrics. In this post, we showed that while adding workers can address certain performance challenges, it’s often not the optimal first response to system bottlenecks.</p> 
<p>Key considerations before scaling workers include:</p> 
<ol> 
 <li>Root cause analysis 
  <ul> 
   <li>Verify whether high CPU/memory usage stems from task optimization issues.</li> 
   <li>Examine if queuing problems result from configuration constraints rather than resource limitations.</li> 
   <li>Investigate potential memory leaks or resource depletion patterns.</li> 
  </ul> </li> 
 <li>Configuration optimization 
  <ul> 
   <li>Review and adjust Airflow parameters (concurrency settings, pools, timeouts).</li> 
   <li>Understand the interaction between different configuration layers.</li> 
   <li>Optimize DAG design and scheduling patterns.</li> 
  </ul> </li> 
</ol> 
<p>The most successful Amazon MWAA implementations follow a systematic approach: first optimizing existing resources and configurations, then scaling workers only when justified by data-driven capacity planning. This approach ensures cost-effective operations while maintaining reliable workflow performance.</p> 
<p>Remember that worker scaling is only one tool in the Amazon MWAA optimization toolkit. Long-term success depends on building a comprehensive performance management strategy that combines proper monitoring, proactive capacity planning, and continuous optimization of your Airflow workflows.</p> 
<p>In the next post, we discuss capacity planning and the steps you need to perform before adding additional DAGs in your environment so that you can plan for the additional load and make sure you have enough headroom.</p> 
<p>To get started, visit the <a href="https://aws.amazon.com/managed-workflows-for-apache-airflow/" rel="noopener noreferrer" target="_blank">Amazon MWAA product page</a> and the <a href="https://docs.aws.amazon.com/mwaa/latest/userguide/best-practices-tuning.html" rel="noopener noreferrer" target="_blank">Performance tuning for Apache Airflow on Amazon MWAA</a> page.</p> 
<p>If you have questions or want to share your MWAA scaling experiences, leave a comment below.</p> 
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
  <p><a href="https://www.linkedin.com/in/ksharlandjiev/" rel="noopener" target="_blank">Kamen</a> is a Principal Big Data and ETL Solutions Architect, Amazon MWAA and AWS Glue ETL expert. He’s on a mission to make life easier for customers who are facing complex data integration and orchestration challenges. His secret weapon? Fully managed AWS services that can get the job done with minimal effort. Follow Kamen on <a href="https://www.linkedin.com/in/ksharlandjiev/" rel="noopener noreferrer" target="_blank">LinkedIn</a> to keep up to date with the latest Amazon MWAA and AWS Glue features and news.</p> 
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
