# Amazon DataZone

Amazon DataZone is a data management service that helps you catalog, discover, govern, share, and analyze your data across your organization and beyond. It enables data producers and consumers to collaborate with built-in governance, data catalog capabilities, and a business data catalog to organize and share data across your AWS environment.

## APIs

### Amazon DataZone API

The Amazon DataZone API provides programmatic access to create and manage data domains, data assets, data catalogs, projects, subscriptions, and governance policies for enterprise-wide data management and sharing.

- **Base URL:** https://datazone.amazonaws.com
- **Documentation:** https://docs.aws.amazon.com/datazone/latest/APIReference/Welcome.html
- **OpenAPI:** [openapi/amazon-datazone-openapi.yml](openapi/amazon-datazone-openapi.yml)

**Tags:** Data Catalog, Data Governance, Data Sharing, Analytics

## Artifacts

| Type | Count | Location |
|------|-------|----------|
| OpenAPI Specs | 1 | [openapi/](openapi/) |
| JSON Schemas | 12 | [json-schema/](json-schema/) |
| JSON Structures | 12 | [json-structure/](json-structure/) |
| Examples | 12 | [examples/](examples/) |
| JSON-LD Contexts | 1 | [json-ld/](json-ld/) |
| Spectral Rules | 1 | [rules/](rules/) |
| Vocabulary | 1 | [vocabulary/](vocabulary/) |
| Naftiko Capabilities | 2 | [capabilities/](capabilities/) |

## Features

- **Business Data Catalog** — Central catalog where data producers publish assets and data consumers can discover, understand, and request access to data products.
- **Domain-Based Governance** — Organize data assets, users, and governance policies within domains that reflect your organizational structure and data ownership.
- **Subscription Workflow** — Built-in request/approval workflow for data consumers to request access to data assets with business justification and audit trail.
- **Project Workspaces** — Isolated project containers within domains where teams organize their data assets, environments, and members.
- **Analytics Environment Provisioning** — Automatically provision data access environments with Athena, Glue, Redshift, or other tools when subscriptions are approved.
- **Glue Data Catalog Integration** — Automatically discover and import tables from AWS Glue Data Catalog into DataZone for cataloging and governance.
- **Data Lineage** — Track data lineage across assets to understand data origins, transformations, and dependencies for trust and compliance.

## Use Cases

- **Enterprise Data Marketplace** — Build an internal data marketplace where business units publish their data products for discovery and consumption by other teams.
- **Data Access Governance** — Implement governed data access with approval workflows ensuring data consumers have proper authorization and business justification.
- **Cross-Account Data Sharing** — Share data assets across AWS accounts within an organization using DataZone's subscription and access management capabilities.
- **Self-Service Analytics** — Enable analysts to discover and access data independently through the DataZone catalog with automatic environment provisioning.
- **Regulatory Data Compliance** — Maintain audit trails of data access, govern sensitive data assets, and enforce data residency policies through domain governance.

## Integrations

- **AWS Glue** — DataZone integrates with Glue Data Catalog to automatically discover, import, and catalog Glue tables for sharing and governance.
- **Amazon Redshift** — Catalog Redshift tables and views and govern cross-cluster data sharing through DataZone subscription workflows.
- **Amazon Athena** — DataZone environments provision Athena as a query engine for subscribers accessing S3-based data assets.
- **Amazon S3** — Catalog S3-based datasets in DataZone and control access through subscription-based Lake Formation permissions.
- **AWS Lake Formation** — DataZone uses Lake Formation for fine-grained column and row-level access control when subscriptions are approved.
- **AWS IAM** — IAM roles provide domain execution context and identity-based access control for DataZone resources and catalog operations.

## Resources

- [Portal](https://aws.amazon.com/datazone/)
- [Documentation](https://docs.aws.amazon.com/datazone/)
- [Getting Started](https://aws.amazon.com/datazone/getting-started/)
- [Pricing](https://aws.amazon.com/datazone/pricing/)
- [FAQ](https://aws.amazon.com/datazone/faqs/)
- [Blog](https://aws.amazon.com/blogs/big-data/)
- [GitHub Organization](https://github.com/aws)
- [Console](https://console.aws.amazon.com/datazone/)
- [Support](https://aws.amazon.com/premiumsupport/)
- [Terms of Service](https://aws.amazon.com/service-terms/)
- [Privacy Policy](https://aws.amazon.com/privacy/)

## Maintainers

- **Kin Lane** — kin@apievangelist.com
