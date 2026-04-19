# Amazon FSx (amazon-fsx)

Amazon FSx provides fully managed file systems with the native compatibility and feature sets for workloads that require shared file storage. FSx supports four widely-used file systems including NetApp ONTAP, OpenZFS, Windows File Server, and Lustre, delivering high performance and low latency access to data.

**URL:** [https://aws.amazon.com/fsx/](https://aws.amazon.com/fsx/)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - AWS, File Systems, Lustre, NetApp, OpenZFS, Storage, Windows

## Timestamps

- **Created:** 2024-01-15
- **Modified:** 2026-04-19

## APIs

### Amazon FSx API
The Amazon FSx API enables programmatic access to create, manage, and monitor fully managed file systems. You can create file systems, manage backups, configure data repositories, create snapshots, and manage storage virtual machines across multiple file system types.

**Human URL:** [https://aws.amazon.com/fsx/](https://aws.amazon.com/fsx/)

#### Tags:

 - File Systems, High Performance, Managed Services, Storage

#### Properties

- [Documentation](https://docs.aws.amazon.com/fsx/latest/APIReference/Welcome.html)
- [OpenAPI](openapi/amazon-fsx-openapi.yml)
- [JSONSchema](json-schema/amazon-fsx-file-system-schema.json)
- [JSONSchema](json-schema/amazon-fsx-backup-schema.json)
- [JSONSchema](json-schema/amazon-fsx-snapshot-schema.json)
- [JSONSchema](json-schema/amazon-fsx-storage-virtual-machine-schema.json)
- [JSONSchema](json-schema/amazon-fsx-tag-schema.json)
- [JSONStructure](json-structure/amazon-fsx-file-system-structure.json)
- [JSONStructure](json-structure/amazon-fsx-backup-structure.json)
- [JSONStructure](json-structure/amazon-fsx-snapshot-structure.json)
- [JSONStructure](json-structure/amazon-fsx-storage-virtual-machine-structure.json)
- [JSONStructure](json-structure/amazon-fsx-tag-structure.json)
- [Example](examples/amazon-fsx-file-system-example.json)
- [Example](examples/amazon-fsx-backup-example.json)
- [Example](examples/amazon-fsx-snapshot-example.json)
- [Example](examples/amazon-fsx-storage-virtual-machine-example.json)
- [Example](examples/amazon-fsx-tag-example.json)
- [GettingStarted](https://aws.amazon.com/fsx/getting-started/)
- [Pricing](https://aws.amazon.com/fsx/pricing/)
- [FAQ](https://aws.amazon.com/fsx/faqs/)
- [APIReference](https://docs.aws.amazon.com/fsx/latest/APIReference/Welcome.html)

## Common Properties

- [Portal](https://aws.amazon.com/fsx/)
- [Website](https://aws.amazon.com/fsx/)
- [Documentation](https://docs.aws.amazon.com/fsx/)
- [TermsOfService](https://aws.amazon.com/service-terms/)
- [PrivacyPolicy](https://aws.amazon.com/privacy/)
- [Support](https://aws.amazon.com/premiumsupport/)
- [Blog](https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/)
- [GitHubOrganization](https://github.com/aws)
- [Console](https://console.aws.amazon.com/fsx/)
- [SignUp](https://portal.aws.amazon.com/billing/signup)
- [StatusPage](https://health.aws.amazon.com/health/status)
- [YouTube](https://www.youtube.com/user/AmazonWebServices)
- [StackOverflow](https://stackoverflow.com/questions/tagged/amazon-fsx)
- [SpectralRules](rules/amazon-fsx-spectral-rules.yml)
- [NaftikoCapability](capabilities/shared/fsx.yaml)
- [NaftikoCapability](capabilities/amazon-fsx-file-system-management.yaml)
- [Vocabulary](vocabulary/amazon-fsx-vocabulary.yaml)
- [JSON-LD](json-ld/amazon-fsx-context.jsonld)

## Features

| Name | Description |
|------|-------------|
| Multiple File System Types | Choose from Lustre, Windows File Server, NetApp ONTAP, and OpenZFS based on workload requirements. |
| High Performance | FSx for Lustre delivers hundreds of GB/s throughput and millions of IOPS for HPC and ML workloads. |
| Native Compatibility | Fully compatible with each file system protocol — SMB for Windows, NFS for Linux, POSIX for Lustre. |
| Automatic Backups | Daily automatic backups stored in Amazon S3 with user-initiated backup support for disaster recovery. |
| Multi-AZ Deployment | FSx for Windows File Server and ONTAP support Multi-AZ configurations for high availability. |
| Data Repository Integration | FSx for Lustre integrates natively with Amazon S3 for transparent data import, export, and auto-release. |
| Encryption at Rest | All file systems are encrypted at rest using AWS KMS with customer-managed key support. |

## Use Cases

| Name | Description |
|------|-------------|
| HPC and ML Training | Use FSx for Lustre for fast scratch storage in high-performance computing and distributed ML training jobs. |
| Windows Workloads | Migrate on-premises Windows file shares to FSx for Windows File Server with Active Directory integration. |
| Enterprise NAS | Use FSx for NetApp ONTAP for enterprise NAS with SnapMirror replication, FlexClone, and multi-protocol access. |
| DevOps and CI/CD | Use FSx for OpenZFS for fast NFS shared storage in development, testing, and containerized workflows. |
| Media Processing | Process high-resolution video and media assets using FSx for Lustre with S3 data repository tiering. |
| Database Backup Storage | Use FSx for Windows File Server or ONTAP as high-performance backup targets for Oracle, SQL Server, and SAP. |

## Integrations

| Name | Description |
|------|-------------|
| Amazon S3 | FSx for Lustre integrates with S3 as a data repository for transparent file import and export. |
| AWS Batch | Use FSx for Lustre as shared scratch storage for parallel AWS Batch compute jobs. |
| Amazon SageMaker | Mount FSx for Lustre directly to SageMaker training instances for fast ML dataset access. |
| Amazon ECS and EKS | Mount FSx volumes as persistent volumes in containerized workloads. |
| AWS Directory Service | Integrate FSx for Windows File Server and ONTAP with Active Directory for user authentication. |
| AWS Backup | Centrally manage FSx backup policies across file systems using AWS Backup. |
| AWS KMS | Encrypt all FSx file systems with customer-managed keys stored in AWS KMS. |
| Amazon CloudWatch | Monitor FSx throughput, IOPS, and latency metrics with CloudWatch dashboards and alarms. |

## Artifacts

Machine-readable API specifications organized by format.

### OpenAPI

- [amazon-fsx-openapi.yml](openapi/amazon-fsx-openapi.yml)

### JSON Schema

- [amazon-fsx-backup-schema.json](json-schema/amazon-fsx-backup-schema.json)
- [amazon-fsx-file-system-schema.json](json-schema/amazon-fsx-file-system-schema.json)
- [amazon-fsx-schema.json](json-schema/amazon-fsx-schema.json)
- [amazon-fsx-snapshot-schema.json](json-schema/amazon-fsx-snapshot-schema.json)
- [amazon-fsx-storage-virtual-machine-schema.json](json-schema/amazon-fsx-storage-virtual-machine-schema.json)
- [amazon-fsx-tag-schema.json](json-schema/amazon-fsx-tag-schema.json)

### JSON Structure

- [amazon-fsx-backup-structure.json](json-structure/amazon-fsx-backup-structure.json)
- [amazon-fsx-file-system-structure.json](json-structure/amazon-fsx-file-system-structure.json)
- [amazon-fsx-snapshot-structure.json](json-structure/amazon-fsx-snapshot-structure.json)
- [amazon-fsx-storage-virtual-machine-structure.json](json-structure/amazon-fsx-storage-virtual-machine-structure.json)
- [amazon-fsx-tag-structure.json](json-structure/amazon-fsx-tag-structure.json)

### JSON-LD

- [amazon-fsx-context.jsonld](json-ld/amazon-fsx-context.jsonld)

### Examples

- [amazon-fsx-backup-example.json](examples/amazon-fsx-backup-example.json)
- [amazon-fsx-file-system-example.json](examples/amazon-fsx-file-system-example.json)
- [amazon-fsx-snapshot-example.json](examples/amazon-fsx-snapshot-example.json)
- [amazon-fsx-storage-virtual-machine-example.json](examples/amazon-fsx-storage-virtual-machine-example.json)
- [amazon-fsx-tag-example.json](examples/amazon-fsx-tag-example.json)

## Capabilities

Naftiko capabilities organized as shared per-API definitions composed into customer-facing workflows.

### Shared Per-API Definitions

- [fsx.yaml](capabilities/shared/fsx.yaml)

### Workflow Capabilities

| Workflow | APIs Combined | Tools | Persona |
|----------|--------------|-------|---------|
| [amazon-fsx-file-system-management.yaml](capabilities/amazon-fsx-file-system-management.yaml) | Amazon FSx API | — | Platform Engineers, DevOps |

## Vocabulary

- [Amazon FSx Vocabulary](vocabulary/amazon-fsx-vocabulary.yaml)

## Rules

- [amazon-fsx-spectral-rules.yml](rules/amazon-fsx-spectral-rules.yml)

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
