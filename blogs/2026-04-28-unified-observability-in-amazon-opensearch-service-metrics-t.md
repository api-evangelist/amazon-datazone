---
title: "Unified observability in Amazon OpenSearch Service: metrics, traces, and AI agent debugging in a single interface"
url: "https://aws.amazon.com/blogs/big-data/unified-observability-in-amazon-opensearch-service-metrics-traces-and-ai-agent-debugging-in-a-single-interface/"
date: "Tue, 28 Apr 2026 17:29:01 +0000"
author: "Muthu Pitchaimani"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p><a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a> now brings application monitoring, native <a href="https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html" rel="noopener noreferrer" target="_blank">Amazon Managed Service for Prometheus</a> integration, and AI agent tracing together in <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/application.html" rel="noopener noreferrer" target="_blank">OpenSearch UI</a>‘s observability workspace. You can query Prometheus metrics with <a href="https://prometheus.io/docs/prometheus/latest/querying/basics/" rel="noopener noreferrer" target="_blank">PromQL</a> alongside logs and traces stored in Amazon OpenSearch Service, trace an AI agent’s full reasoning chain down to the failing tool call, and drill from a service-level health view to the exact span that caused a checkout failure, all without leaving the interface.</p> 
<p>In this post, we walk through two real-world scenarios using the OpenTelemetry sample app: a multi-agent travel planner facing slow processing, and a checkout flow quietly failing on one microservice. We chase each one to its root cause using these new capabilities.</p> 
<h2>Scenario 1: An underperforming AI agent</h2> 
<p>Your multi-agent travel planner is live and users start reporting slow responses. With the new AI agent tracing capability in Amazon OpenSearch Service, you can trace the agent’s full processing path to pinpoint exactly where things went wrong.</p> 
<p>In any observability workspace in OpenSearch UI, navigate to <strong>Application Map</strong> in the left navigation pane.</p> 
<p><img alt="OpenSearch Service application map" class="alignnone size-full wp-image-90438" height="1520" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image003.jpg" width="2258" /></p> 
<p>You can see the full topology of your system including the travel agent and the sub-agents it calls. The travel agent node shows elevated latency and occasional errors. Select it, and the side panel confirms that latency is up but the latency chart shows intermittent spikes rather than consistent degradation.</p> 
<p><img alt="System topology with service health metrics" class="alignnone size-full wp-image-90439" height="1302" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image005-scaled.jpg" width="2560" /></p> 
<p>The application map tells you something is wrong, but understanding <em>why</em> an AI agent is underperforming requires seeing its reasoning chain. Select <strong>Agent Traces</strong> in the left navigation pane, then filter by service name and time range.</p> 
<p><img alt="Agent processing steps with invocation data" class="alignnone size-full wp-image-90440" height="728" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image007.png" width="1430" /></p> 
<p>Select one of the traces to see the trace tree. Unlike a traditional span waterfall, this view organizes around the agent’s reasoning chain: the root agent span, the LLM calls it made, the tools it invoked, and how they nested each step color-coded by type. The trace map provides a visual directed graph of the same execution. You can see which model was called, how many input and output tokens were consumed, and the actual messages sent to and received from the model.</p> 
<p>A tool call inside the weather agent errored out. The agent then spent additional time reasoning about the failure before returning a partial response explaining the intermittent latency spikes and occasional faults.</p> 
<h3>Why this matters for AI agents</h3> 
<p>Agents make autonomous decisions based on LLM responses, tool results, and chained reasoning. Unlike traditional microservices with deterministic code paths, agent behavior varies across executions. Without semantic tracing that captures these AI-specific signals, root-cause analysis is guesswork. The trace tree surfaced the model name, token counts, and failing tool call because the travel planner was instrumented with OpenTelemetry’s generative AI semantic conventions. The next section describes how.</p> 
<h3>Instrumenting AI agents</h3> 
<p>OpenTelemetry auto-instrumentation enriches spans with well-known attributes for HTTP, database, and gRPC calls. AI agents need a different set of attributes such as which LLM was called, what tokens were consumed, which tools were invoked, that standard instrumentation doesn’t cover.</p> 
<p>The <a href="https://opentelemetry.io/docs/specs/semconv/gen-ai/" rel="noopener" target="_blank">OpenTelemetry gen_ai semantic conventions</a> define standard attributes for these signals, including <code>gen_ai.operation.name</code>, <code>gen_ai.usage.input_tokens</code>, <code>gen_ai.request.model</code>, and <code>gen_ai.tool.name</code>. When Amazon OpenSearch Service receives spans with these attributes, it categorizes them by operation type (agent, LLM, tool, embeddings, retrieval) and renders the agent trace tree and trace map views.</p> 
<p>The Python SDK provides one way to generate these spans. To send traces to Amazon OpenSearch Ingestion, configure the SDK with AWS Signature Version 4 (SigV4) authentication. The <code>AWSSigV4OTLPExporter</code> cryptographically signs each HTTP request to help prevent unauthorized data ingestion. The calling identity needs an IAM policy that grants <code>osis:Ingest</code> on your pipeline’s ARN. Credentials are resolved through the standard AWS credential provider chain.</p> 
<pre><code class="language-python">from opensearch_genai_observability_sdk_py import register, AWSSigV4OTLPExporter

exporter = AWSSigV4OTLPExporter(
    endpoint="https://pipeline.us-east-1.osis.amazonaws.com/v1/traces",
    service="osis",
    region="us-east-1",
)

register(service_name="my-agent", exporter=exporter)
</code></pre> 
<p>Use the <code>@observe</code> decorator to trace agent functions and <code>enrich()</code> to add model metadata:</p> 
<pre><code class="language-python">@observe(op=Op.EXECUTE_TOOL)
def get_weather(city: str) -&gt; dict:
    return {"city": city, "temp": 22, "condition": "sunny"}

@observe(op=Op.INVOKE_AGENT)
def assistant(query: str) -&gt; str:
    enrich(model="gpt-4o", provider="openai")
    data = get_weather("Paris")
    return f"{data['condition']}, {data['temp']}C"

result = assistant("What's the weather?")
</code></pre> 
<p>The SDK also supports auto-instrumentation for OpenAI, Anthropic, Amazon Bedrock, LangChain, LlamaIndex, and others. Because the instrumentation is built on OpenTelemetry standards, any agent framework that emits spans with <code>gen_ai.*</code> attributes is compatible with OpenSearch UI.</p> 
<h2>Scenario 2: Investigating a microservice issue</h2> 
<p>AI agents are only one part of most production environments. The same interface surfaces telemetry from conventional microservices, where the troubleshooting workflow follows a more familiar path.</p> 
<p>Your ecommerce checkout begins paging during a busy traffic window. From OpenSearch UI, navigate to <strong>APM Services</strong> in the left navigation pane. Every instrumented service is listed alongside its health indicators. The checkout service shows an elevated error rate.</p> 
<p><img alt="Service overview panel with request, error, duration metrics" class="alignnone size-full wp-image-90441" height="1306" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image009-scaled.jpg" width="2560" /></p> 
<p>Select the affected service. The detail view shows Request, Error, and Duration (RED) metrics: request rate is climbing, fault rate has spiked in the last 15 minutes, and p99 duration has doubled. You can see exactly when the degradation started.</p> 
<p><img alt="Service drilldown health dashboard" class="alignnone size-full wp-image-90442" height="723" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image011.png" width="1431" /></p> 
<p>Drill into the correlated spans for the affected time window. The span list shows multiple failed requests, all hitting the same endpoint. Select one to see the full trace waterfall. The checkout service called <code>prepareOrder</code>, which failed trying to retrieve a product from the catalog. The error message in the span details tells you exactly what went wrong, that’s your root cause.</p> 
<p><img alt="Waterfall transaction view of spans" class="alignnone wp-image-90443 size-full" height="730" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image013.png" width="1429" /></p> 
<h3>Checking the infrastructure with PromQL</h3> 
<p>In both scenarios, the natural next question is whether the problem originates in the application or in the infrastructure beneath it. With the new Amazon Managed Service for Prometheus integration, you can answer that question without leaving OpenSearch UI.</p> 
<p>Prometheus metrics are now queryable directly from the same workspace using native PromQL syntax, alongside the logs and traces you’ve already been navigating.</p> 
<p><img alt="Metric query showing Prometheus Query Language" class="alignnone size-full wp-image-90444" height="820" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image015.png" width="1431" /></p> 
<p>For the database timeout in Scenario 2, run a PromQL query to check the database instance’s read/write throughput for the same time window. For the agent latency issue in Scenario 1, check the LLM endpoint’s response time metrics to see if the slowness originates from the model provider.</p> 
<p>This is a key architectural decision: metrics continue to live in Amazon Managed Service for Prometheus, logs and traces continue to live in Amazon OpenSearch Service, and neither signal is copied or warehoused into a second store. Each backend remains the single store for the data type it’s purpose-built to handle, while OpenSearch UI federates queries across both at runtime. The cost, retention, and operational model of each store stay intact while the troubleshooting workflow collapses into a single interface.</p> 
<p>To configure the OpenTelemetry Collector and OpenSearch Ingestion pipelines that route metrics into Amazon Managed Service for Prometheus, see <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/observability-ingestion.html" rel="noopener" target="_blank">Ingesting application telemetry</a>.</p> 
<h2>How it’s wired together</h2> 
<p>The following diagram shows the end-to-end architecture. Applications instrumented with OpenTelemetry send traces, logs, and metrics over OTLP to Amazon OpenSearch Ingestion. OpenSearch Ingestion routes each signal to the appropriate store: traces and logs land in Amazon OpenSearch Service, while metrics flow into Amazon Managed Service for Prometheus. OpenSearch UI then queries both stores to render the Application Map, Services catalog, Agent Traces, and Metrics views.</p> 
<p><img alt="OpenSearch Observability Stack Architecture" class="alignnone size-full wp-image-90446" height="472" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image019.png" width="1202" /></p> 
<p>The entire experience rests on open-source foundations, Prometheus for metrics, OpenSearch for logs and traces, and OpenTelemetry for instrumentation, so teams already running an OpenTelemetry collector can adopt it by updating the collector’s export configuration to point at Amazon OpenSearch Ingestion, with no proprietary agents or rewritten instrumentation required.</p> 
<h2>Getting started</h2> 
<p>To enable these capabilities, log in to OpenSearch UI’s observability workspace, select the <strong>Gear</strong> icon in the bottom left corner to open Settings and setup, and verify that the <strong>Observability:apmEnabled</strong> toggle is on under the Observability section. OpenSearch UI is available at no additional charge for Amazon OpenSearch Service customers.</p> 
<div class="wp-video" style="width: 640px;">
 <video class="wp-video-shortcode" controls="controls" height="360" id="video-90656-1" preload="metadata" width="640">
  <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/BDB-5856/BDB-5856.mp4?_=1" type="video/mp4" />
 </video>
</div> 
<p><strong>Explore locally first.</strong> The <a href="https://opensearch.org/platform/observability-stack/" rel="noopener" target="_blank">OpenSearch Observability Stack</a> gives you a fully configured environment including application monitoring, agent tracing, and Prometheus integration, running on your machine with a single install command. It ships with sample instrumented services, including a multi-agent travel planner, so you can explore the full workflow with real telemetry data out of the box.</p> 
<p><strong>For AI agent development.</strong> <a href="https://observability.opensearch.org/docs/agent-health/" rel="noopener" target="_blank">Agent Health</a> is an open-source, evaluation-driven observability tool designed for local development. It gives you execution flow graphs, token tracking, and tool invocation visibility right in your development loop, before you push to production.</p> 
<p><strong>For production.</strong> The <a href="https://observability.opensearch.org/docs/send-data/ai-agents/python/" rel="noopener" target="_blank">Python SDK</a> provides one-line setup and decorator-based tracing with gen_ai semantic conventions, with auto-instrumentation support for OpenAI, Anthropic, Amazon Bedrock, LangChain, LlamaIndex, and others. See the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/observability.html" rel="noopener" target="_blank">Amazon OpenSearch Service documentation</a> and the <a href="https://docs.aws.amazon.com/opensearch-service/latest/developerguide/direct-query-prometheus-overview.html" rel="noopener" target="_blank">Amazon Managed Service for Prometheus integration guide</a> for the full managed experience.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-90447" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image021.png" width="100" />
  </div> 
  <h3 class="lb-h4">Muthu Pitchaimani</h3> 
  <p>Muthu is a Search Specialist with Amazon OpenSearch Service. He builds large-scale search applications and solutions. Muthu is interested in the topics of networking and security, and is based out of Austin, Texas.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-90450" height="102" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image022.png" style="font-size: 16px;" width="100" />
  </div> 
  <h3 class="lb-h4">Raaga N.G</h3> 
  <p><a href="https://www.linkedin.com/in/raaga-shree/" rel="noopener noreferrer" target="_blank">Raaga</a> is a Solutions Architect at AWS with over 5 years of experience helping enterprises modernize their technology landscape and build scalable, cloud-native solutions. She partners with customers to translate business requirements into efficient cloud architectures that drive measurable outcomes, supporting their journey from application modernization to AI adoption through thoughtful, customer-centric solutions.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-90448" height="2560" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image023.png" width="1920" />
  </div> 
  <h3 class="lb-h4">Rekha Thottan</h3> 
  <p>Rekha Thottan is a Senior Technical Product Manager at AWS OpenSearch, contributing to AI agent observability and evaluation for the OpenSearch Project.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-90449" height="768" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/20/image025.png" width="576" />
  </div> 
  <h3 class="lb-h4">Kevin Lewin</h3> 
  <p>Kevin is a Cloud Operations Specialist Solution Architect at Amazon Web Services. He focuses on helping customers achieve their operational goals through observability and automation.</p> 
 </div> 
</footer>
