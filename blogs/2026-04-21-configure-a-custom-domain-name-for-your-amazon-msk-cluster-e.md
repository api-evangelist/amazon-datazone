---
title: "Configure a custom domain name for your Amazon MSK cluster enabled with IAM authentication"
url: "https://aws.amazon.com/blogs/big-data/configure-a-custom-domain-name-for-your-amazon-msk-cluster-enabled-with-iam-authentication/"
date: "Tue, 21 Apr 2026 16:33:29 +0000"
author: "Mazrim Mehrtens"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Most <a href="https://aws.amazon.com/msk/" rel="noopener noreferrer" target="_blank">Amazon Managed Streaming for Apache Kafka</a> (Amazon MSK) customers are simplifying and standardizing access control to Kafka resources using&nbsp;<a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a>&nbsp;(IAM) authentication. This adoption is also accelerated as <a href="https://aws.amazon.com/blogs/big-data/amazon-msk-iam-authentication-now-supports-all-programming-languages/" rel="noopener noreferrer" target="_blank">Amazon MSK now supports IAM authentication in popular languages</a> including&nbsp;<a href="https://github.com/aws/aws-msk-iam-auth" rel="noopener noreferrer" target="_blank">Java</a>, <a href="https://github.com/aws/aws-msk-iam-sasl-signer-python" rel="noopener noreferrer" target="_blank">Python</a>, <a href="https://github.com/aws/aws-msk-iam-sasl-signer-go" rel="noopener noreferrer" target="_blank">Go</a>, <a href="https://github.com/aws/aws-msk-iam-sasl-signer-js" rel="noopener noreferrer" target="_blank">JavaScript</a>, and <a href="https://github.com/aws/aws-msk-iam-sasl-signer-net" rel="noopener noreferrer" target="_blank">.NET</a>.</p> 
<p>In the first part of <a href="https://aws.amazon.com/blogs/big-data/configure-a-custom-domain-name-for-your-amazon-msk-cluster/" rel="noopener noreferrer" target="_blank">Configure a custom domain name for your Amazon MSK cluster</a>, we discussed about why custom domain names are important and provided details on how to configure a custom domain name in Amazon MSK when using SASL_SCRAM authentication. In this post, we discuss how to configure a custom domain name in Amazon MSK when using IAM authentication. We recommend you read the first part of this blog as it captures solution details implementation steps.</p> 
<h2>Solution overview</h2> 
<p>IAM authentication for Amazon MSK uses TLS to encrypt the Kafka protocol traffic between the client and Kafka broker. To use a custom domain name, the Kafka broker needs to present a server certificate that matches the custom domain name. To achieve this, this solution uses an&nbsp;<a href="https://aws.amazon.com/elasticloadbalancing/network-load-balancer/" rel="noopener noreferrer" target="_blank">Network Load Balancers (NLBs)</a> with Amazon Certificate Manager to provide a custom certificate on behalf of the MSK brokers, and a Route 53 Private Hosted Zone to provide DNS for the custom domain name.</p> 
<p>The following diagram shows all components used by the solution.</p> 
<p><img alt="Architecture showing configuration of custom domain name with Amazon MSK" class="size-full wp-image-89068 aligncenter" height="536" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-1-2.jpg" width="571" /></p> 
<h3>Certificate management</h3> 
<p>For clients to perform TLS communication with the MSK cluster the cluster needs to provide a certificate with hostnames matching the custom domain name. This solution uses a certificate in&nbsp;<a href="https://aws.amazon.com/certificate-manager/" rel="noopener noreferrer" target="_blank">AWS Certificate Manager</a> (ACM) signed with a Private Certificate Authority (PCA) for TLS with the custom domain name. This solution uses a&nbsp;certificate with&nbsp;<code>bootstrap.example.com</code> as the Common Name (CN) so that the certificate is valid for the bootstrap address, and Subject Alternative Names (SANs) are set for all broker DNS names (such as&nbsp;<code>b-1.example.com</code>). Since this solution uses a private certificate authority, the CA chain must be imported into the client trust stores.</p> 
<p>This solution works with any server certificate, whether certificates are signed by a public or private Certificate Authority (CA). You can import existing certificates into ACM to be used with this solution. Certificates must provide a common name and/or subject alternative names that match the bootstrap DNS address as well as the individual broker DNS addresses. If the certificate is issued by a private CA, clients need to import the root and intermediate CA certificates to the client trust store. If the certificate is issued by a public CA, the root and intermediate CA certificates will be in the default trust store.</p> 
<h3>Network Load Balancer</h3> 
<p>The NLB provides the ability to use a <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-tls-listener.html" rel="noopener noreferrer" target="_blank">TLS listener</a>. The ACM certificate is associated with the listeners and enables TLS negotiation between the client and the NLB. The NLB performs a separate TLS negotiation between itself and the MSK brokers. In addition to the above architecture, this solution also allows using AWS Private Link to connect the cluster to external VPCs. This allows secure access to MSK between VPCs while using a custom domain name.</p> 
<p>The following diagram illustrates the NLB port and target configuration. A TLS listener with port 9000 is used for bootstrap connections with all MSK brokers set as targets. IAM authentication is configured to run on port 9098 of the MSK brokers using a TLS target type. A TLS listener port is used to represent each broker in the MSK cluster. In this post, there are three brokers in the MSK cluster starting with port 9001, representing broker 1 and up to port 9003, representing broker 3.</p> 
<p><img alt="Target Group mapping in NLB" class="size-full wp-image-89067 aligncenter" height="748" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-2-2.jpg" width="1001" /></p> 
<h3>Domain Name System (DNS)</h3> 
<p>For the client to resolve DNS queries for the custom domain, we use an <a href="https://aws.amazon.com/route53/" rel="noopener noreferrer" target="_blank">Amazon Route 53</a> <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html" rel="noopener noreferrer" target="_blank">private hosted zone</a> to host the DNS records, and associate it with the client’s VPC to enable DNS resolution from the <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html" rel="noopener noreferrer" target="_blank">Route 53 VPC resolver</a>. This solution uses a private MSK cluster and private DNS. For publicly accessible MSK clusters a public NLB and DNS provider such as a <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/AboutHZWorkingWith.html">Route53 public hosted zone</a> can be used.</p> 
<h3>Amazon MSK</h3> 
<p>Finally, each broker needs to have its advertised listeners configuration (<code>advertised.listeners</code>) updated to match the custom domain name and NLB ports.&nbsp;Advertised listeners is a configuration option used by Kafka clients to connect to the brokers. By default, an advertised listener is not set. Once set, Kafka clients use the advertised listener instead of&nbsp;<code>listeners</code> to obtain the connection information for brokers.&nbsp;MSK brokers use the listener configuration to tell clients the DNS names and ports to use to connect to the individual brokers for each authentication type enabled. Advertised listeners are unique to each broker; and the cluster won’t start if multiple brokers have the same advertised listener address. For this reason, this solution uses a unique custom DNS name for each broker&nbsp;(such as,&nbsp;<code>b-1.example.com</code>).</p> 
<h2>Solution Deployment</h2> 
<p>To deploy the solution, use the CloudFormation template from the <a href="https://github.com/aws-samples/sample-msk-custom-domain-name-iam-auth" rel="noopener noreferrer" target="_blank">GitHub</a> repository.</p> 
<p>This template deploys a VPC, NLB, PCA, ACM certificate, MSK cluster, and an Amazon EC2 instance for cluster connectivity. The EC2 instance includes a script to handle updating the broker <code>advertised.listeners</code> settings to match the custom domain name. For more information on deploying a CloudFormation template, refer to&nbsp;<a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html">Create a stack from the CloudFormation console</a>.</p> 
<p>After deploying the CloudFormation template, run the script to update advertised listeners as follows:</p> 
<ol> 
 <li>Retrieve the <strong>MSKClusterARN</strong> and <strong>CertificateAuthorityARN</strong> from the CloudFormation outputs for your stack as they will be used in subsequent steps.<br /> <img alt="" class="size-full wp-image-89066 aligncenter" height="1052" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-3-4.png" width="2324" /></li> 
 <li>Navigate to the EC2 console and identify the KafkaClientInstance. Choose <strong>Connect</strong> to connect to the instance using <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html" rel="noopener noreferrer" target="_blank">AWS Systems Manager Session Manager</a>.</li> 
 <li>Session Manager starts a session in shell. Start a bash session with the command: 
  <div class="hide-language"> 
   <pre><code class="lang-shell">bash -l</code></pre> 
  </div> <p><img alt="" class="alignnone size-full wp-image-89913" height="149" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/07/msk-iam-domain-image-4.jpg" width="656" /></p></li> 
 <li>The Kafka client SDKs have already been installed in the EC2 instance. You can update the <code>advertised.listeners</code> configuration as follows, replacing <strong>CLUSTER_ARN</strong> with the ARN of your MSK cluster retrieved from CloudFormation in step 1: 
  <div class="hide-language"> 
   <pre><code class="lang-shell">./update_advertised_listeners.sh --region us-east-1 --cluster-arn CLUSTER_ARN</code></pre> 
  </div> <p>Note that once this script completes, the brokers will have new advertised listeners configurations. Connections using the standard IAM address for the MSK service will not work until we complete the next steps, as the brokers will redirect connections over this address back to the custom domain name and TLS will fail.</p></li> 
 <li>Next, we need to create a truststore with the certificate for our AWS Private Certificate Authority (PCA) to allow TLS with the NLB. In the following command, replace <strong>PCA_ARN</strong> with the ARN of the PCA retrieved from CloudFormation in step 1:<br /> <img alt="" class="size-full wp-image-89064 aligncenter" height="836" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-5-3.png" width="2324" />We’re using the default Java truststore which uses the password <code>changeit</code>.When asked “Trust this certificate?” enter “yes”.<p></p> 
  <div class="hide-language"> 
   <pre><code class="lang-shell">export&nbsp;PCA_ARN=&lt;&lt;PCA_ARN&gt;&gt;
export&nbsp;REGION=&lt;&lt;REGION&gt;&gt;

cp /etc/pki/java/cacerts . &amp;&amp; chmod 600 cacerts
aws acm-pca get-certificate-authority-certificate --certificate-authority-arn $PCA_ARN --region $REGION&nbsp;| jq -r '.Certificate' &gt; pca.pem
keytool -import -file pca.pem -alias AWSPCA -keystore&nbsp;cacerts</code></pre> 
  </div> </li> 
 <li>Create a new properties file to allow IAM authentication with our custom truststore: 
  <div class="hide-language"> 
   <pre><code class="lang-shell">cat &lt;&lt;EOF &gt;&gt; /home/ssm-user/client-iam.properties
ssl.truststore.location=/home/ssm-user/cacerts
ssl.truststore.password=changeit
EOF</code></pre> 
  </div> </li> 
 <li>Verify you can connect to the cluster using IAM authentication using our new custom domain name, replacing bootstrap.example.com with your own custom domain name if you used a different one in CloudFormation: 
  <div class="hide-language"> 
   <pre><code class="lang-code">bin/kafka-topics.sh --list --command-config client-iam.properties --bootstrap-server bootstrap.example.com:9000</code></pre> 
  </div> <p><img alt="" class="alignnone wp-image-89544 size-full" height="360" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/26/bdb5167i6.jpg" width="2560" /></p></li> 
</ol> 
<h2>Cleanup</h2> 
<p>To stop incurring costs navigate to CloudFormation and delete the CloudFormation stack to remove all resources provisioned by CloudFormation.</p> 
<h2>Frequently Asked Question about Custom Domain Name</h2> 
<p>Customers have asked a few questions about implementing custom domain names with MSK. You can find answers to some of the most popular questions here.</p> 
<h3>Are there any limitations for this solution on MSK?</h3> 
<p>The <code>advertised.listeners</code> setting was removed as a dynamic broker in KRaft-based Kafka clusters. Therefore, this solution is only supported in Zookeeper-based MSK clusters. Additionally, this solution is only applicable to SASL/SCRAM and IAM-authentication based MSK clusters.</p> 
<h3>How the custom domain name solution scales when we add new brokers?</h3> 
<p>When using the NLB for broker connectivity (<a href="https://aws.amazon.com/blogs/big-data/configure-a-custom-domain-name-for-your-amazon-msk-cluster/#:~:text=Option%202%3A%20All%20connections%20through%20an%20NLB" rel="noopener noreferrer" target="_blank">option 2 in the configure a custom domain name for your Amazon MSK cluster blog post</a>), you will need to add an additional listener for each additional broker created.</p> 
<p>For TLS, if using Subject Alternative Name (SAN) to list individual broker DNS hostnames, you will need to create a new certificate that includes the names of the additional brokers. One option is to create a certificate with SANs for more brokers than needed to allow for growth.If a wildcard certificate is used, you do not need to modify certificates when adding brokers.</p> 
<h3>What changes are required when we remove brokers?</h3> 
<p>Amazon MSK supports scale-in by removing brokers from the cluster. Brokers are removed from each availability zones (AZ). So a 6 broker Amazon MSK cluster deployed in 3 AZ can be reduced to 3 broker cluster deployed in 3 AZ. When brokers are removed, you can remove the NLB listeners for the removed broker along with the Route53 DNS endpoints. However, you can also leave them as is, or just remove the target IP from the broker numbers target group. The NLB will mark the targets as unhealthy and stop directing traffic to them. If you ever plan to scale-out the number of brokers, you can re-use the existing NLB listeners and Route 53 DNS entries and would only need to update the target IPs used in the broker numbers target group.</p> 
<h3>Is there any change in configuration required if there is any broker failure?</h3> 
<p>No. When a broker fails, Amazon MSK replaces the failed broker with a new broker instance keeping the configuration of the broker exactly the same. So, there would be no change in the advertised listener of the broker. Once the broker is healthy, the broker can accept new connections and read/write traffic.</p> 
<p><strong>Can you use Amazon MSK Replicator between MSK clusters in multiple AWS Regions when using the custom domain name solution?</strong></p> 
<p>The <a href="https://aws.amazon.com/msk/features/msk-replicator/" rel="noopener noreferrer" target="_blank">Amazon MSK Replicator</a> can be used when using the custom domain name solution, either in an active-passive or active-active setup. The same process can be followed to set the custom domain name.</p> 
<p>You then follow <a href="https://aws.amazon.com/blogs/big-data/build-multi-region-resilient-apache-kafka-applications-with-identical-topic-names-using-amazon-msk-and-amazon-msk-replicator/" rel="noopener noreferrer" target="_blank">build multi-Region resilient Apache Kafka applications with identical topic names using Amazon MSK and Amazon MSK Replicator</a> post to configure MSK Replicator.</p> 
<p>The following diagram shows an active-active AWS multi-Region MSK setup using the custom domain name solution:</p> 
<p><img alt="" class="size-full wp-image-89062 aligncenter" height="823" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-7-5.png" width="1430" /></p> 
<p><strong>Can I use a global bootstrap DNS name to connect to Amazon MSK clusters deployed across multiple AWS regions when IAM authentication is enabled?</strong></p> 
<p>No, it is not possible to use a global bootstrap reference to represent MSK clusters deployed in multiple AWS Regions, unless the client is aware of the cluster’s region when connecting. To use IAM authentication, the correct AWS Region must be included in the IAM authentication request for a given cluster. This is because the AWS Region is a part of the Sigv4 authentication protocol used by IAM. This scope prevents the IAM authorization being used to talk to a resource in another AWS Region. You can provide the AWS Region in one of two ways– with region-specific bootstrap URLs or by explicitly configuring the region.</p> 
<p>For example, if the bootstrap string is <a href="http://bootstrap.us-east-1.example.com/" rel="noopener noreferrer" target="_blank">bootstrap.us-east-1.example.com</a>, then <a href="https://github.com/aws/aws-msk-iam-auth" rel="noopener noreferrer" target="_blank">msk-iam-auth</a> library will to extract the AWS Region from the broker connection string and use us-east-1 in its IAM requests. If the bootstrap string is simply <a href="http://bootstrap.example.com/" rel="noopener noreferrer" target="_blank">bootstrap.example.com</a>, then the client must explicitly configure AWS_REGION=us-east-1 to connect to the cluster if it is in us-east-1, or us-west-2 if it is in us-west-2.</p> 
<p>Note that this is a limitation for IAM authentication, but not for SASL/SCRAM authentication. With SASL/SCRAM authentication, if the client’s credentials are applied to both clusters the global endpoint can point to either cluster and the client will be able to connect. The AWS Region is not used in SASL/SCRAM authentication, so it does not restrict the authentication scope.</p> 
<h3>How to allow public access to a private MSK cluster using the custom domain name solution?</h3> 
<p>To provide public access to a MSK cluster using the custom domain solution, you will need to do the following:</p> 
<ul> 
 <li>Create an Internet-facing NLB, and associate public subnets (subnets that have a route to the Internet Gateway attached to the VPC).</li> 
 <li>Create ingress rules in both the NLB and MSK security groups permitting the required public addresses. Note: the port will be 9098 for the MSK security group, and the ports you are using on the NLB listeners.</li> 
 <li>Provide public DNS resolution for the Kafka clients, by using a Route 53 public zone, or an alternative public DNS resolver.</li> 
 <li>The client needs have IAM credentials, with permission, to talk to the MSK brokers, using an <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" rel="noopener noreferrer" target="_blank">IAM role</a>,&nbsp;<a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/security-creds-programmatic-access.html" rel="noopener noreferrer" target="_blank">IAM access keys</a>, <a href="https://aws.amazon.com/iam/roles-anywhere/" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere</a>, or another mechanism that uses the AWS Security Token Service (AWS STS) to create and provide trusted users with temporary security credentials.</li> 
</ul> 
<p><img alt="" class="size-full wp-image-89061 aligncenter" height="1262" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-8.jpg" width="1478" /></p> 
<h3>In the first part of the blog, two patterns have been highlighted.&nbsp;How to decide which pattern to use and why?</h3> 
<h3>Option 1: Only bootstrap connection through NLB</h3> 
<p>If the Kafka clients have direct access to the broker, then you can use custom domain name for the bootstrap connection while the clients can still connect to the MSK Brokers with broker DNS. This is the simplest option, as it does not require custom TLS certificates or TLS listeners.Note that this option is not necessary when using MSK Express brokers, as MSK Express brokers already manages bootstrapping via a broker-agnostic connection string. For MSK Express, this option does not add value other than configuring a custom domain name for appearances / simplicity of client configuration. For MSK Standard brokers, this can improve client connectivity by making connection strings broker agnostic.</p> 
<p><img alt="" class="size-full wp-image-89060 aligncenter" height="107" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-9.jpg" width="625" /></p> 
<h3>Option 2:&nbsp;All connections through NLB</h3> 
<p>When Kafka clients don’t have direct access to Amazon MSK Brokers, routing all connections through the NLB can be preferred. This can occur when a client is deployed in a different VPC than Amazon MSK VPC or the client is external, and when Amazon MSK Multi VPC Connectivity is not an option. In general, Amazon MSK Multi VPC Connectivity is preferred as this is a simpler pattern for most organizations to manage MSK Connectivity across accounts and VPCs.When Multi VPC Connectivity is not an option, NLB can be used to provide connectivity with Transit Gateway or PrivateLink, and the solution mentioned in the blog should be used.</p> 
<p><img alt="" class="size-full wp-image-89059 aligncenter" height="168" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-10.jpg" width="623" /></p> 
<p>Here is an example architecture how Kafka client and Amazon MSK cluster deployed in two separate VPCs but connected via AWS Private Link.</p> 
<p><img alt="" class="size-full wp-image-89058 aligncenter" height="611" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/18/image-11-2.png" width="1287" /></p> 
<h3>Is Amazon Route 53 required to use a custom domain name with Amazon MSK?</h3> 
<p>You can use an alternative DNS resolver service, and do not require Amazon Route 53 to use a custom domain name with Amazon MSK. The only requirement is that your clients can resolve against your DNS resolver service. The only change required, is to use a CNAME for the DNS records, referencing the <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html#dns-name" rel="noopener noreferrer" target="_blank">NLBs DNS record</a>, in place of the Alias records, as this is record type is only available in Amazon Route 53.</p> 
<h3>We don’t use Amazon Certificate Manager (ACM), can NLB integrate with other 3rd party certificate managers?</h3> 
<p>NLB only supports ACM to bind a certificate to a TLS listener. You can import a certificate created using your 3rd party certificate manager into ACM, and do not need to create a certificate using ACM.</p> 
<h3>Getting connection to node terminated during authentication after setting&nbsp;<code>advertised.listeners</code>&nbsp;, what could be the issue?</h3> 
<p>As the issue started to occur after changing the&nbsp;<code>advertised.listeners</code>&nbsp;configuration, the issue is unlikely to be related to permissions. The following can cause this issue:</p> 
<ul> 
 <li>The NLB and/or client’s Security Group does not permit access to the listener ports on the NLB from the client.</li> 
 <li>A firewall appliance between the NLB and client does not permit the client to talk to the NLB using the listener ports.</li> 
 <li>The&nbsp;<code>advertised.listeners</code>&nbsp;configuration has an error causing the client to receive invalid details, such as a typo in the name. If this is the case, use a client in the same VPC as the MSK broker that has IAM permissions to talk to the MSK broker, and Security Group rules permitting connectivity, you then use the following command to delete the&nbsp;<code>advertised.listeners</code>&nbsp;configuration.</li> 
</ul> 
<div class="hide-language"> 
 <pre><code class="lang-shell">/home/ec2-user/kafka/bin/kafka-configs.sh --alter \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --bootstrap-server  \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-type brokers \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-name  \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --command-config ~/kafka/config/client_iam.properties \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --delete-config advertised.listeners</code></pre> 
</div> 
<p>BROKERS_AMAZON_DNS_NAME such as&nbsp;<code>b-1.clustername.xxxxxx.yy.kafka.region.amazonaws.com:9098</code>.</p> 
<h3>Getting “unexpected broker id, expected 2 or empty string, but received 1”, what is causing this error?</h3> 
<p>This error is typically presented when the&nbsp;<code>advertised.listeners</code>&nbsp;configuration for one of the brokers has the port used by another broker set. For example broker 2 has port 9001 set for IAM, but this port is used to connect to broker 1, so broker 1 is responding with an error to say you presented broker id 2, but I am broker 1.</p> 
<p>To correct this, you will need to update the broker with the incorrect&nbsp;<code>advertised.listeners</code>&nbsp;configuration to use the correct port. To gain access to the broker to make the change, you will need to use the following command to delete the incorrect configuration:</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">/home/ec2-user/kafka/bin/kafka-configs.sh --alter \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --bootstrap-server \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-type brokers \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-name  \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --command-config ~/kafka/config/client_iam.properties \
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --delete-config advertised.listeners</code></pre> 
</div> 
<p>BROKERS_AMAZON_DNS_NAME such as&nbsp;<code>b-2.clustername.xxxxxx.yy.kafka.region.amazonaws.com:9098</code>.</p> 
<p>You then need to use the following command to set the&nbsp;<code>advertised.listeners</code>&nbsp;configuration for that broker:</p> 
<p>Note:&nbsp;The&nbsp;<code>advertised.listeners</code>&nbsp;configuration in the below assumes only IAM is used for authentication. If you are using additional authentication options, you will need to include them.</p> 
<div class="hide-language"> 
 <pre><code class="lang-shell">MSKDOMAIN=
broker_id=
Domain=

/home/ec2-user/kafka/bin/kafka-configs.sh&nbsp;--alter&nbsp;\
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --bootstrap-server&nbsp;&nbsp;\
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-type&nbsp;brokers&nbsp;\
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --entity-name&nbsp;"$broker_id"&nbsp;\
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --command-config&nbsp;~/kafka/config/client_iam.properties&nbsp;\
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; --add-config&nbsp;"advertised.listeners=[CLIENT_IAM://b-$broker_id.$Domain:900$broker_id,REPLICATION://b-$broker_id-internal.$MSKDOMAIN:9093,REPLICATION_SECURE://b-$broker_id-internal.$MSKDOMAIN:9095]"</code></pre> 
</div> 
<h2>Summary</h2> 
<p>In this post, we explained how you can use an NLB, Route 53, and the advertised listener configuration option in Amazon MSK to support custom domain names with MSK clusters when using IAM authentication. You can use this solution to keep your existing Kafka bootstrap DNS name and reduce or remove the need to change client applications because of a migration, recovery process, or to use a DNS name in line with your organization’s naming convention (for example,&nbsp;msk.prod.example.com).</p> 
<p>Try the solution out for yourself, and leave your questions and feedback in the comments section.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Subham Rakshit" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2024/04/26/subham.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Subham Rakshit</h3> 
  <p><a href="https://www.linkedin.com/in/subhamrakshit/" rel="noopener" target="_blank">Subham</a> is a Senior Streaming Solutions Architect for Analytics at AWS based in the UK. He works with customers to design and build streaming architectures so they can get value from analyzing their streaming data. His two little daughters keep him occupied most of the time outside work, and he loves solving jigsaw puzzles with them.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Mark Taylor" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2024/06/17/mgtaylor_headshot.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Mark Taylor</h3> 
  <p><a href="https://www.linkedin.com/in/mark-taylor-5b77a525/" rel="noopener" target="_blank">Mark</a> is a Senior Technical Account Manager at AWS, working with enterprise customers to implement best practices, optimize AWS usage, and address business challenges. Mark lives in Folkestone, England, with his wife and two dogs. Outside of work, he enjoys watching and playing football, watching movies, playing board games, and traveling.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignnone size-full wp-image-89475" height="107" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/03/24/bdb-5775-mmehrten-headshot.png" width="100" />
  </div> 
  <h3 class="lb-h4">Mazrim Mehrtens</h3> 
  <p><a href="https://www.linkedin.com/in/mmehrtens/" rel="noopener" target="_blank">Mazrim</a> is a Sr. Specialist Solutions Architect for messaging and streaming workloads. Mazrim works with customers to build and support systems that process and analyze terabytes of streaming data in real time, run enterprise Machine Learning pipelines, and create systems to share data across teams seamlessly with varying data toolsets and software stacks.</p> 
 </div> 
</footer>
