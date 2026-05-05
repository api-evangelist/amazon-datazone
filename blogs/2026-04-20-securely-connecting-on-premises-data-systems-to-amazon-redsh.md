---
title: "Securely connecting on-premises data systems to Amazon Redshift with IAM Roles Anywhere"
url: "https://aws.amazon.com/blogs/big-data/securely-connecting-on-premises-data-systems-to-amazon-redshift-with-iam-roles-anywhere/"
date: "Mon, 20 Apr 2026 14:59:23 +0000"
author: "Zainab Syeda"
feed_url: "https://aws.amazon.com/blogs/big-data/feed/"
---
<p>Securely connecting on-premises data systems to <a href="https://aws.amazon.com/redshift/" rel="noopener noreferrer" target="_blank">Amazon Redshift</a> requires removing static credentials while preserving seamless access for your data teams. This solution extends connectivity from your on-premises data centers to Amazon Redshift by using short-lived, auditable credentials. All traffic remains within trusted, private channels.</p> 
<p>Developers and data engineers need a process to run ingestion pipelines, Extract, Transform, Load (ETL) jobs, and analytics queries without managing static credentials or complex authentication flows. You can use <a href="https://aws.amazon.com/iam/roles-anywhere/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM) Roles Anywhere</a> to obtain temporary security credentials in IAM. This service extends the short-term credential model of AWS beyond the cloud and allows on-premises workloads to authenticate with IAM using X.509 certificates from an existing certificate authority. This approach removes static IAM access keys and applies least-privilege access through IAM policies. Every request is recorded in <a href="https://aws.amazon.com/cloudtrail/" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a>. Paired with private Domain Name System (DNS) and Amazon Virtual Private Cloud (Amazon VPC) endpoints for Amazon Redshift, it keeps authentication and data flows inside private networks without traversing the public internet.</p> 
<p>In this post, you will learn how to use AWS IAM Roles Anywhere with Amazon Redshift for secure, private connections. This removes the need to expose traffic to the public internet or manage long-lived access keys.</p> 
<h2>The challenge</h2> 
<p>Organizations connecting on-premises data systems to Amazon Redshift typically choose from several established security patterns, each with tradeoffs in risk, complexity, and operational overhead. Static IAM access keys are straightforward to adopt but require ongoing rotation, secure distribution, and storage across systems. Their long-lived nature increases the impact of accidental exposure in code, configuration files, or logs. Shared database or service credentials can streamline setup but often reduce auditability, weaken least-privilege controls, and create accountability challenges across teams. VPN or private network connections improve network isolation, yet they still require strong application-layer authentication and add infrastructure management burdens. Custom secret-management or credential-brokering solutions can reduce reliance on long-lived credentials, but they introduce additional components that must be built, integrated, and maintained. As organizations scale, these patterns often force tradeoffs between strong security controls and the developer productivity needed to build and operate data pipelines efficiently.</p> 
<h2>Solution overview</h2> 
<p>The solution integrates on-premises workloads with Amazon Redshift using IAM Roles Anywhere and the built-in IAM authentication of Amazon Redshift. The core idea is that on-premises workloads use X.509 certificates to obtain short-term IAM credentials, then exchange them for temporary Amazon Redshift database credentials. Both provisioned clusters and serverless workgroups are supported. The architecture consists of these main components:</p> 
<ul> 
 <li><strong>Amazon Redshift Service Endpoint</strong> – Handles secure API calls such as <a href="https://docs.aws.amazon.com/redshift/latest/APIReference/API_GetClusterCredentials.html" rel="noopener noreferrer" target="_blank">GetClusterCredentials</a>, <a href="https://docs.aws.amazon.com/redshift-serverless/latest/APIReference/API_GetCredentials.html" rel="noopener noreferrer" target="_blank">GetCredentials</a>, and <a href="https://docs.aws.amazon.com/redshift/latest/APIReference/API_GetClusterCredentialsWithIAM.html" rel="noopener noreferrer" target="_blank">GetClusterCredentialsWithIAM</a>. The on-premises workload uses these API endpoints to request temporary database credentials.</li> 
 <li><strong>Amazon Redshift Cluster Endpoint</strong> – Provides the connection point for database operations on provisioned Amazon Redshift clusters. After obtaining temporary credentials, applications and tools like JDBC/ODBC drivers or psql connect to the cluster endpoint. They use this connection to execute SQL queries, load data, and perform analytics tasks.</li> 
 <li><strong>Amazon Redshift Serverless Workgroup Endpoint </strong>– Serves the same function as the cluster endpoint but for serverless deployments. After temporary credentials are retrieved through the&nbsp;GetCredentials&nbsp;API, applications connect to this endpoint using standard database drivers (JDBC/ODBC) or command line tools like psql to run queries and load data.</li> 
 <li><strong>Certificate authority</strong>&nbsp;– For this post, we use&nbsp;<a href="https://aws.amazon.com/private-ca/" rel="noopener noreferrer" target="_blank">AWS Private Certificate Authority (AWS Private CA)</a>&nbsp;as the certificate authority (CA) source. Alternatively, you can integrate with an external CA. For more details, see&nbsp;<a href="https://aws.amazon.com/blogs/security/iam-roles-anywhere-with-an-external-certificate-authority/" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere with an external certificate authority</a>.</li> 
 <li><strong>X.509 Certificate&nbsp;</strong>– We use a sample&nbsp;<a href="https://docs.aws.amazon.com/acm/latest/userguide/private-certificates.title.html" rel="noopener noreferrer" target="_blank">private certificate</a>&nbsp;stored in&nbsp;<a href="https://aws.amazon.com/certificate-manager/" rel="noopener noreferrer" target="_blank">AWS Certificate Manager (ACM)</a>&nbsp;and issued by AWS Private CA.</li> 
 <li><strong>IAM Roles Anywhere</strong> – Issues short-term AWS credentials to on-premises processes based on X.509 certificates from an organization’s certificate authority. These temporary credentials allow the workload to assume an IAM role that grants access to Amazon Redshift APIs.</li> 
</ul> 
<p>To retrieve temporary credentials using IAM Role Anywhere, we use the&nbsp;<code>credential_process</code>&nbsp;parameter in <a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI) profile configurations to trigger an <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html" rel="noopener noreferrer" target="_blank">external process</a> that generates or retrieves credentials. This post uses <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication.html" rel="noopener noreferrer" target="_blank">X.509 certificates to authenticate</a> and return temporary IAM credentials through IAM Roles Anywhere. The <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html" rel="noopener noreferrer" target="_blank">AWS IAM Roles Anywhere Credential Helper</a> is executed to handle the <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication-sign-process.html" rel="noopener noreferrer" target="_blank">signing process</a> for the <a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication-create-session.html" rel="noopener noreferrer" target="_blank">CreateSession</a> API, returning credentials in a JSON format that applications and tools can consume.</p> 
<p>Amazon Redshift provides several APIs that work together to support temporary, IAM-based authentication for different deployment scenarios. When connecting to a&nbsp;provisioned Amazon Redshift cluster, applications typically use the&nbsp;<code>GetClusterCredentials</code>&nbsp;API, which returns short-term database credentials tied to an IAM role’s permissions. For organizations with fully IAM-managed identities,&nbsp;<code>GetClusterCredentialsWithIAM</code>&nbsp;streamlines this process by automatically mapping the IAM identity to a database user, removing the need to specify usernames manually. In&nbsp;serverless deployments, the&nbsp;<code>GetCredentials</code>&nbsp;API performs the same function, issuing temporary credentials for Amazon Redshift Serverless workgroups based on IAM permissions. Collectively, these APIs keep static credentials from being stored or distributed while offering flexible integration paths for both provisioned and serverless Amazon Redshift architectures.</p> 
<h3>Flow overview</h3> 
<p>An on-premises ETL job begins by initiating a request and authenticates with AWS using IAM Roles Anywhere to assume an IAM role securely. After obtaining temporary security credentials, the workload calls the Amazon Redshift service endpoint to execute the <code>GetClusterCredentials</code> API, which returns short-term database credentials. These credentials allow the workload to connect to the Amazon Redshift cluster endpoint through a VPC endpoint. This enables running SQL queries or loading data into the cluster as part of the ETL process.</p> 
<p><img alt="" class="alignnone size-full wp-image-90157" height="580" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/14/BDB-5469-1-1.png" width="1010" /></p> 
<h2>Prerequisites</h2> 
<p><strong>You must have the following prerequisites to follow along with this post.</strong></p> 
<p>AWS account requirements</p> 
<ul> 
 <li>An AWS account with permissions to deploy&nbsp;<a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a>&nbsp;templates.</li> 
 <li>Access to&nbsp;<a href="https://aws.amazon.com/cloudshell/" rel="noopener noreferrer" target="_blank">AWS CloudShell</a>&nbsp;for exporting a sample private certificate that we create using AWS CloudFormation in a later step.</li> 
</ul> 
<p>Remote environment</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html" rel="noopener noreferrer" target="_blank">AWS CLI version 2</a></li> 
 <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere Credential Helper tool</a></li> 
</ul> 
<p><a href="https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/hybrid-connectivity.html" rel="noopener noreferrer" target="_blank">Network Connectivity</a> requirements</p> 
<ul> 
 <li>Establish secure connectivity between your on-premises environment and AWS using <a href="https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html" rel="noopener noreferrer" target="_blank">AWS Site-to-Site VPN</a>, <a href="https://aws.amazon.com/directconnect/" rel="noopener noreferrer" target="_blank">AWS Direct Connect</a>, or <a href="https://aws.amazon.com/vpn/client-vpn/" rel="noopener noreferrer" target="_blank">AWS Client VPN</a>.</li> 
</ul> 
<h2>Deploy AWS resources with AWS CloudFormation</h2> 
<ol> 
 <li>Navigate to the&nbsp;<a href="https://console.aws.amazon.com/cloudformation" rel="noopener noreferrer" target="_blank">AWS CloudFormation console</a>.</li> 
 <li>Choose&nbsp;<strong>Create Stack.</strong></li> 
 <li>Download the&nbsp;<a href="https://github.com/aws-samples/sample-redshift-iamra-template/blob/main/RedshiftIAMRolesAnywhere.yaml" rel="noopener noreferrer" target="_blank">redshift-iamra-template</a> template.</li> 
 <li>For&nbsp;<strong>Specify template,&nbsp;</strong>choose&nbsp;<strong>Upload a template file&nbsp;</strong>and upload <a href="https://github.com/aws-samples/sample-redshift-iamra-template/blob/main/RedshiftIAMRolesAnywhere.yaml" rel="noopener noreferrer" target="_blank">redshift-iamra-template</a>.</li> 
 <li>Choose&nbsp;<strong>Next</strong>.</li> 
 <li>Enter a unique name for&nbsp;<strong>Stack name</strong>. The default value is&nbsp;<code>redshift-test</code>.</li> 
 <li>Configure the stack parameters. The following table provides default values.</li> 
</ol> 
<table border="1px" cellpadding="10px" class="styled-table" style="height: 1072px;" width="757"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Parameter name</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Default value</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>VPCCIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10.0.0.0/16</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CIDR block for the VPC</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PrivateSubnet1CIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10.0.1.0/24</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CIDR block for the first private subnet</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>PrivateSubnet2CIDR</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">10.0.2.0/24</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">CIDR block for the second private subnet</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CACommonName</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">redshift-ca.example.com</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Common Name for the Certificate</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CAOrganization</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Example Corp</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Organization for the Certificate Authority</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CACountry</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">US</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Country for the Certificate Authority</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CAValidityInDays</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1826</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Validity period in days for the CA Certificate (5 years)</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftClusterIdentifier</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-redshift-cluster</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Identifier for the Amazon Redshift cluster</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftDatabaseName</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>dev</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Name of the initial database in the Amazon Redshift cluster</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftMasterUsername</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>admin</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Main username for the Amazon Redshift cluster</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftNodeType</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ra3.xlplus</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Node type for the Amazon Redshift cluster</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ServerlessNamespace</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-serverless-namespace</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Namespace identifier for Amazon Redshift Serverless</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ServerlessWorkgroup</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-serverless-workgroup</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Workgroup identifier for Amazon Redshift Serverless</td> 
  </tr> 
 </tbody> 
</table> 
<ol start="8"> 
 <li>Select the <strong>acknowledgement</strong> checkbox and choose&nbsp;<strong>Create Stack</strong>. Stack deployment takes about 10 minutes to complete.</li> 
</ol> 
<ol start="9"> 
 <li>When stack creation is complete, navigate to the&nbsp;<strong>Outputs</strong>&nbsp;tab on the AWS CloudFormation console and note down the values for the resources that the stack created.</li> 
</ol> 
<p>The following table shows a summarized view of the output values.</p> 
<table border="1px" cellpadding="10px" class="styled-table" style="height: 988px;" width="984"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Output</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Description</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Example value</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>CertificateAuthorityArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name (ARN)</a> of the Private Certificate Authority</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:acm-pca:aa-example-1:111122223333:certificate-authority/a1b2c3d4-5678-90ab-cdef-EXAMPLE22222</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ClientCertificateArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the sample client certificate</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:acm:aa-example-1:111122223333:certificate/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ProfileArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the IAM Roles Anywhere profile</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:rolesanywhere:aa-example-1:111122223333:profile/a1b2c3d4-5678-90ab-cdef-EXAMPLE44444</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftAccessRoleArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the Amazon Redshift Access role</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:iam::1222345677:role/Redshift-test-RedshiftAccessRole</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>TrustAnchorArn</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ARN of the IAM Roles Anywhere profile. You will use this value for configuring&nbsp;<code>credential_process</code>&nbsp;for IAM Roles Anywhere in a later step.</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>arn:aws:rolesanywhere:aa-example-1:111122223333:trust-anchor/a1b2c3d4-5678-90ab-cdef-EXAMPLE33333</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftClusterEndpoint</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Private endpoint of the Amazon Redshift Cluster</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-redshift-cluster-123456789012.aa-example-1.redshift.amazonaws.com</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>RedshiftClusterPort</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Port of the Amazon Redshift Cluster</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>5439</code></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>ServerlessWorkgroupEndpoint</code></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Private endpoint of Amazon Redshift Serverless Workgroup</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><code>my-serverless-workgroup-123456789012.aa-example-1.redshift.serverless.amazonaws.com</code></td> 
  </tr> 
 </tbody> 
</table> 
<h2>Export a sample private certificate using CloudShell</h2> 
<p>To export a sample private certificate using CloudShell, complete the following steps.</p> 
<ol> 
 <li>Open CloudShell. For more details, see&nbsp;<a href="https://docs.aws.amazon.com/cloudshell/latest/userguide/working-with-aws-cloudshell.html#navigating-the-interface" rel="noopener noreferrer" target="_blank">Navigating the AWS CloudShell interface</a>.</li> 
 <li>Export the certificate ARN from the CloudFormation outputs. If you changed the stack name in the previous step, use that value for&nbsp;<code>&lt;stack-name&gt;</code>. Otherwise, use the default value&nbsp;<code>redshift-public-iam-roles-anywhere</code>.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">export CERT_ARN=$(aws cloudformation describe-stacks \
    --stack-name &lt;stack-name&gt; \
    --query 'Stacks[0].Outputs[?OutputKey==`ClientCertificateArn`].OutputValue' \
    --output text)
</code></pre> 
</div> 
<ol start="3"> 
 <li>Extract the certificate and private key files:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-shell"># Generate and save the passphrase
export PASSPHRASE=$(openssl rand -base64 32)
# Export certificate using environment variables
aws acm export-certificate \
    --certificate-arn $CERT_ARN \
    --passphrase $(echo -n "$PASSPHRASE" | base64) \
    &gt; cert_export.json
# Extract components to separate files
jq -r '.Certificate' cert_export.json &gt; certificate.pem
jq -r '.PrivateKey' cert_export.json &gt; encrypted_private_key.pem
# Decrypt the private key
openssl rsa -in encrypted_private_key.pem -out private_key.pem -passin pass:"$PASSPHRASE"
# Clear environment variables
unset PASSPHRASE CERT_ARN</code></pre> 
</div> 
<ol start="4"> 
 <li><a href="https://docs.aws.amazon.com/cloudshell/latest/userguide/getting-started.html#download-file" rel="noopener noreferrer" target="_blank">Download</a>&nbsp;the extracted certificate and private key files from CloudShell:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">/home/cloudshell-user/certificate.pem
/home/cloudshell-user/private_key.pem</code></pre> 
</div> 
<ol start="5"> 
 <li>Secure the private key on your local workstation.</li> 
</ol> 
<p>After downloading the files, restrict file permissions to prevent unauthorized access:</p> 
<pre><code>chmod 400 private_key.pem</code> <code>chmod 400 certificate.pem</code></pre> 
<p>For production workloads, consider storing private keys in your operating system’s keychain (macOS Keychain, Windows Certificate Store), a hardware security module (HSM), or a secrets management tool rather than as files on disk.</p> 
<h2>Configure an AWS CLI profile</h2> 
<p>These are the steps to configure an AWS CLI profile on your system:</p> 
<ol> 
 <li>Store the downloaded certificate and private key to your environment. For an automated approach to generate and rotate certificates, see&nbsp;<a href="https://aws.amazon.com/blogs/security/set-up-aws-private-certificate-authority-to-issue-certificates-for-use-with-iam-roles-anywhere/" rel="noopener noreferrer" target="_blank">Set up AWS Private Certificate Authority to issue certificates for use with IAM Roles Anywhere</a>.</li> 
 <li>Create a new profile named&nbsp;<code>onprem-redshift</code>. This invokes the credential process. Replace the placeholders with your specific values. Find the values for&nbsp;<code>trusted-anchor-arn</code>,&nbsp;<code>profile-arn</code>, and&nbsp;<code>role-arn</code>&nbsp;in your CloudFormation stack outputs.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-html">aws configure set profile.onprem-redshift.credential_process "&lt;/path/to/aws_signing_helper&gt; credential-process \
      --certificate &lt;/path/to/certificate.pem&gt; \
      --private-key &lt;/path/to/private_key.pem&gt; \
      --trust-anchor-arn &lt;trusted-anchor-arn&gt; \
      --profile-arn &lt;profile-arn&gt; \
      --role-arn &lt; role-arn&gt;"</code></pre> 
</div> 
<ol start="3"> 
 <li>Verify your configuration. Open the&nbsp;<code>~/.aws/config</code>&nbsp;file and confirm that it contains a profile.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-html">[profile onprem-redshift]
credential_process = &lt;/path/to/aws_signing_helper&gt; credential-process       
--certificate &lt;/path/to/certificate.pem&gt;       
--private-key &lt;/path/to/private_key.pem&gt;       
--trust-anchor-arn &lt;trusted-anchor-arn&gt;       
--profile-arn &lt;profile-arn&gt;       
--role-arn &lt;role-arn&gt;</code></pre> 
</div> 
<h2>Test the solution</h2> 
<p>Follow these steps to validate your setup for provisioned clusters to confirm end-to-end connectivity:</p> 
<ol> 
 <li>Verify network connectivity</li> 
</ol> 
<p>Before testing authentication, confirm that your on-premises environment can reach the Amazon Redshift cluster endpoint:</p> 
<p><code>telnet my-redshift-cluster.abc123.us-east-1.redshift.amazonaws.com 5439</code></p> 
<p>If the connection succeeds, you should see a response indicating the port is open. If it fails, verify your VPN/Direct Connect configuration and security group rules.</p> 
<ol start="2"> 
 <li>Create database user</li> 
</ol> 
<p>If you haven’t already created a user, connect to your Amazon Redshift as the main user and create a dedicated user for testing:</p> 
<p><code>CREATE USER analytics_user PASSWORD '[PASSWORD]';</code></p> 
<ol start="3"> 
 <li>Retrieve Amazon Redshift database credentials</li> 
</ol> 
<p>With the configuration in place, request temporary database credentials from Amazon Redshift:</p> 
<div class="hide-language"> 
 <pre><code class="lang-powershell">aws redshift get-cluster-credentials \
  --db-user analytics_user \
  --cluster-identifier my-redshift-cluster \
  --region us-east-1 \
  --profile onprem-redshift</code></pre> 
</div> 
<p>This call returns a short-lived username and password that’s valid for connecting to the cluster. By default, the temporary credentials expire in 900 seconds. You can optionally specify a duration between 900–3600 seconds (15–60 minutes).</p> 
<ol start="4"> 
 <li>Connect using JDBC/ODBC or psql</li> 
</ol> 
<p>Use the issued credentials in your connection string. For JDBC:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">jdbc:redshift://my-redshift-cluster.abc123.redshift.amazonaws.com:5439/dev?ssl=true&amp;UID=analytics_user&amp;PWD=&lt;temporary_password&gt;</code></pre> 
</div> 
<p>For psql:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">PGPASSWORD=&lt;temporary_password&gt; psql \
  -h my-redshift-cluster.abc123.redshift.amazonaws.com \
  -p 5439 \
  -U analytics_user \
  -d dev \
  --set=sslmode=verify-full</code></pre> 
</div> 
<h3>Validate and monitor</h3> 
<ul> 
 <li>Test authentication flows end-to-end using your ETL jobs.</li> 
 <li>Review AWS CloudTrail logs to validate. It records role assumptions and Amazon Redshift API calls.</li> 
 <li>Monitor session expiration to help workloads handle credential refresh seamlessly.</li> 
</ul> 
<h3>Testing end-to-end connectivity for Amazon Redshift Serverless</h3> 
<p>The testing process for Amazon Redshift Serverless follows a similar pattern to provisioned clusters, with minor differences in the API calls and connection parameters. These steps validate connectivity to your serverless workgroup.</p> 
<ol> 
 <li>Verify network connectivity</li> 
</ol> 
<p><code>telnet my-serverless-workgroup.abc123.us-east-1.redshift.amazonaws.com 5439</code></p> 
<ol start="2"> 
 <li>Retrieve Amazon Redshift Serverless database credentials</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-powershell">aws redshift-serverless get-credentials \
  --workgroup-name my-serverless-workgroup \
  --db-name dev \
  --region us-east-1 \
  --profile onprem-redshift</code></pre> 
</div> 
<ol start="3"> 
 <li>Connect using JDBC/ODBC or psql</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-code">PGPASSWORD="&lt;password_from_get_credentials&gt;" psql \
  -h my-serverless-workgroup.abc12.us-east-1.redshift-serverless.amazonaws.com \
  -p 5439 \
  -U "IAMR:Redshift-IAMRA-RedshiftAccessRole" \
  -d dev \
  --set=sslmode=verify-full</code></pre> 
</div> 
<h2>Clean up</h2> 
<p>To avoid future charges, remove the deployed resources:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html" rel="noopener noreferrer" target="_blank">Delete</a> the CloudFormation stack.</li> 
 <li>Remove the generated files from CloudShell:</li> 
</ol> 
<p><code>rm cert_export.json encrypted_private_key.pem certificate.pem private_key.pem</code></p> 
<h2>Conclusion</h2> 
<p>In this post, we showed how to implement IAM Roles Anywhere with Amazon Redshift so that enterprises can securely connect on-premises data systems to their cloud data warehouse without relying on static credentials or public internet access. This architecture provides short-lived, auditable credentials, integrates with existing certificate authorities, and helps ensure authentication and data flows remain private and trusted.</p> 
<p>With this approach, data engineers and developers can run ingestion pipelines, ETL jobs, and analytics queries, while security teams maintain full control through IAM governance and CloudTrail auditing. You can remove manual credential rotation tasks, allow your data engineers to connect to Amazon Redshift without managing static keys, and achieve complete audit trails through CloudTrail integration for your hybrid analytics environments.</p> 
<p>To get started, deploy the solution using the&nbsp;<a href="https://github.com/aws-samples/sample-redshift-iamra-template/blob/main/RedshiftIAMRolesAnywhere.yaml" rel="noopener noreferrer" target="_blank">CloudFormation template</a>&nbsp;and follow the steps in this post. To learn more about the services used, see the following resources:</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html" rel="noopener noreferrer" target="_blank">IAM Roles Anywhere documentation</a></li> 
 <li><a href="https://docs.aws.amazon.com/redshift/latest/mgmt/generating-user-credentials.html" rel="noopener noreferrer" target="_blank">Amazon Redshift IAM authentication</a></li> 
 <li><a href="https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html" rel="noopener noreferrer" target="_blank">AWS Private Certificate Authority</a></li> 
 <li><a href="https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html" rel="noopener noreferrer" target="_blank">AWS CloudTrail for auditing</a></li> 
</ul> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignleft wp-image-90158 size-thumbnail" height="101" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/14/BDB-5469-2-1-100x101.png" width="100" />
  </div> 
  <p><a href="https://www.linkedin.com/in/k-bajwa/" rel="noopener noreferrer" target="_blank">Kanwar</a> Bajwa is a Principal Enterprise Account Engineer at AWS who works with customers to optimize their use of AWS services and achieve their business objectives.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignleft wp-image-90159 size-thumbnail" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/14/BDB-5469-3-1-100x133.jpeg" width="100" />
  </div> 
  <p><a href="https://www.linkedin.com/in/xiaoxuexu/" rel="noopener noreferrer" target="_blank">Xiaoxue</a> Xu is a Solutions Architect for AWS based in Toronto. She primarily works with Financial Services customers to help secure their workload and design scalable solutions on the AWS Cloud.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="alignleft wp-image-90156 size-thumbnail" height="133" src="https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2026/04/14/BDB-5469-4-100x133.png" width="100" />
  </div> 
  <p><a href="https://www.linkedin.com/in/syeda-zainab/" rel="noopener noreferrer" target="_blank">Zainab</a> Syeda&nbsp;is a Technical Account Manager at Amazon Web Services in Toronto. She works with customers in the Financial Services segment, helping them leverage cloud-native solutions at scale.</p> 
 </div> 
</footer>
