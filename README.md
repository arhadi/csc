# IAM Policies & Roles Documentation (DWH)

This document provides a comprehensive overview of all IAM Roles and Policies used across the OMRON Data Warehouse (DWH) architecture on AWS, including AWS Account mapping, execution privileges, and exact S3 bucket access lists partitioned by environment (`dev`, `preprod`, `prod`).

---

## 1. AWS Account Mapping & Architecture Overview

Based on the target data lake architecture:
- **`awsap-common-stage-account`**: Hosts both **DEV** and **PREPROD** environments.
- **`awsap-common-prod-account`**: Hosts the **PROD** environment.

```
[Data Sources (NEXUS, GCRM, ECRM, APPIAS)]
                   │
                   ▼ (Ingestion)
         [Landing S3 Buckets]
                   │
                   ▼ (Glue ETL Jobs & Transformations)
         [Staging S3 Buckets]
                   │
                   ▼ (Snowflake Storage Integration)
     [Snowflake Accounts (OEP, OEB, ARATAS)]
```

---

## 2. Roles, Target Accounts, & Execution Privileges Matrix

| Service | Environment | Directory Path | Trusted Principal (AssumeRole) | Primary Purpose & Execution Capabilities |
| :--- | :--- | :--- | :--- | :--- |
| **AWS Glue** | `dev` | `iam/glue/dev/` | `glue.amazonaws.com` | • Read data from S3 Landing Dev<br>• Write transformed data to S3 Staging Dev<br>• Access AWS Secrets Manager (database credentials)<br>• Connect to VPC/Subnets via ENI (JDBC JDE connectivity)<br>• **Send email alerts/reports via Amazon SES** |
| **AWS Glue** | `preprod` | `iam/glue/preprod/` | `glue.amazonaws.com` | • Same capabilities as dev, with exact resources targeted to **Preprod** |
| **AWS Glue** | `prod` | `iam/glue/prod/` | `glue.amazonaws.com` | • Production ETL execution: read Landing Prod, write Staging Prod, access Secrets Manager Prod, send email alerts via SES |
| **MWAA (Airflow)** | `dev` | `iam/mwaa/dev/` | `airflow.amazonaws.com`<br>`airflow-env.amazonaws.com` | • Execute Airflow DAGs on MWAA<br>• Read & write DAGs and plugin assets (`sst-s3-*-mwaa-dags`)<br>• Monitor Landing & Staging Dev buckets (Airflow S3 Sensors)<br>• Trigger Glue ETL jobs (`glue:StartJobRun`)<br>• Send pipeline notifications via SES |
| **MWAA (Airflow)** | `preprod` | `iam/mwaa/preprod/` | `airflow.amazonaws.com`<br>`airflow-env.amazonaws.com` | • Same capabilities as dev, orchestrating pipelines for **Preprod** |
| **MWAA (Airflow)** | `prod` | `iam/mwaa/prod/` | `airflow.amazonaws.com`<br>`airflow-env.amazonaws.com` | • **Production** pipeline orchestration: trigger Glue Prod jobs, monitor S3 Prod, send alerts via SES |
| **S3 Role** | `dev` | `iam/s3/dev/` | `root` / Snowflake IAM User | • **Snowflake Storage Integration**: Read S3 Staging Dev (OEP, OEB, ARATAS) for Snowflake ingestion<br>• **Data Ingestion**: Ingest source records into S3 Landing Dev<br>• KMS S3 decryption |
| **S3 Role** | `preprod` | `iam/s3/preprod/` | `root` / Snowflake IAM User | • Snowflake Storage Integration for Staging Preprod & Ingestion into Landing Preprod |
| **S3 Role** | `prod` | `iam/s3/prod/` | `root` / Snowflake IAM User | • Snowflake Storage Integration for Staging Prod & Ingestion into Landing Prod |
| **SES Role** | `dev` / `preprod` / `prod` | `iam/ses/<env>/` | `glue.amazonaws.com`<br>`airflow.amazonaws.com`<br>`lambda.amazonaws.com`<br>`root` | • Dedicated role for sending email alerts via Amazon SES<br>• Publish delivery/bounce events to Amazon SNS (`SNS -> Email`) |
| **Terraform Deployment** | `dev` / `preprod` / `prod` | `iam/terraform/<env>/` | CI/CD Deployer Role (`sts:AssumeRole`) | • Automated infrastructure provisioning: S3, KMS, IAM, VPC Endpoints, Secrets, Glue Jobs/Connections, MWAA, Lambda |

---

## 3. S3 Bucket Access Matrix per Role & Environment

The table below outlines the exact S3 bucket resources each service role is authorized to read from and write to:

| Service Role | Env | S3 Read (Source) | S3 Write (Destination) | Script / DAGs Bucket |
| :--- | :--- | :--- | :--- | :--- |
| **Glue** | `dev` | `datalake-oepoeb-dev-landing`<br>`datalake-aratas-dev-landing` | `datalake-oep-dev-staging`<br>`datalake-oeb-dev-staging`<br>`datalake-aratas-dev-staging` | `sst-s3-*-mwaa-dags` |
| **Glue** | `preprod` | `datalake-oepoeb-preprod-landing`<br>`datalake-aratas-preprod-landing` | `datalake-oep-preprod-staging`<br>`datalake-oeb-preprod-staging`<br>`datalake-aratas-preprod-staging` | `sst-s3-*-mwaa-dags` |
| **Glue** | `prod` | `datalake-oepoeb-prod-landing`<br>`datalake-aratas-prod-landing` | `datalake-oep-prod-staging`<br>`datalake-oeb-prod-staging`<br>`datalake-aratas-prod-staging` | `sst-s3-*-mwaa-dags` |
| **MWAA** | `dev` | `datalake-oepoeb-dev-landing`<br>`datalake-*-dev-staging` | - | `sst-s3-*-mwaa-dags` (Read & Write) |
| **MWAA** | `preprod` | `datalake-oepoeb-preprod-landing`<br>`datalake-*-preprod-staging` | - | `sst-s3-*-mwaa-dags` (Read & Write) |
| **MWAA** | `prod` | `datalake-oepoeb-prod-landing`<br>`datalake-*-prod-staging` | - | `sst-s3-*-mwaa-dags` (Read & Write) |
| **S3 (Snowflake / Ingest)** | `dev` | `datalake-oep-dev-staging`<br>`datalake-oeb-dev-staging`<br>`datalake-aratas-dev-staging` | `datalake-oepoeb-dev-landing`<br>`datalake-aratas-dev-landing` | - |
| **S3 (Snowflake / Ingest)** | `preprod` | `datalake-oep-preprod-staging`<br>`datalake-oeb-preprod-staging`<br>`datalake-aratas-preprod-staging` | `datalake-oepoeb-preprod-landing`<br>`datalake-aratas-preprod-landing` | - |
| **S3 (Snowflake / Ingest)** | `prod` | `datalake-oep-prod-staging`<br>`datalake-oeb-prod-staging`<br>`datalake-aratas-prod-staging` | `datalake-oepoeb-prod-landing`<br>`datalake-aratas-prod-landing` | - |

---

## 4. Policy JSON Directory Structure

Every service directory contains a standardized pair of JSON policy documents per environment:
- `iam_policy.json`: The permission policy granting actions on specified resources.
- `trust_policy.json`: The AssumeRole trust policy determining authorized principals.

```
config/dwh/iam/
├── glue/
│   ├── dev/       (iam_policy.json, trust_policy.json)
│   ├── preprod/   (iam_policy.json, trust_policy.json)
│   └── prod/      (iam_policy.json, trust_policy.json) - alias: prd/
├── mwaa/
│   ├── dev/       (iam_policy.json, trust_policy.json)
│   ├── preprod/   (iam_policy.json, trust_policy.json)
│   └── prod/      (iam_policy.json, trust_policy.json) - alias: prd/
├── s3/
│   ├── dev/       (iam_policy.json, trust_policy.json)
│   ├── preprod/   (iam_policy.json, trust_policy.json)
│   └── prod/      (iam_policy.json, trust_policy.json) - alias: prd/
├── ses/
│   ├── dev/       (iam_policy.json, trust_policy.json)
│   ├── preprod/   (iam_policy.json, trust_policy.json)
│   └── prod/      (iam_policy.json, trust_policy.json) - alias: prd/
└── terraform/
    ├── dev/       (iam_policy.json, trust_policy.json)
    ├── preprod/   (iam_policy.json, trust_policy.json)
    └── prod/      (iam_policy.json, trust_policy.json) - alias: prd/
```

---

## 5. Technical Implementation Notes

> [!NOTE]
> **Direct SES Execution from AWS Glue**:
> Permissions to dispatch emails via SES (`ses:SendEmail`, `ses:SendRawEmail`, `ses:SendTemplatedEmail`) are embedded **directly** inside Glue's `iam_policy.json` for all environments (`dev`, `preprod`, `prod`). This ensures that Python scripts executed in AWS Glue (`boto3.client('ses')`) can immediately deliver job completion or failure alert emails using Glue's execution role credentials without requiring an additional `sts:AssumeRole` step.

> [!TIP]
> **AWS Lambda Integration Standby**:
> `lambda.amazonaws.com` is retained in the SES trust policies and Terraform deployment policies as a *standby permission*. Should Lambda functions be introduced into the ingestion or alerting workflows in the future, the IAM architecture is fully prepared with zero downtime or policy refactoring required.

---

## 6. Attribute-Based Access Control (ABAC) & Resource Tag Conditions

To adhere to enterprise security standards and the principle of least privilege, all Data Lake service policies employ **Attribute-Based Access Control (ABAC)** using AWS IAM `Condition` blocks. 

Access is governed by a **Defense-in-Depth** model: both the **explicit resource ARN** and the **AWS Resource Tags** (`aws:ResourceTag/env` and `aws:ResourceTag/appname`) must match for a request to be authorized.

### 6.1 Tag Specification

| Tag Key | Allowed Values | Applied To Resources |
| :--- | :--- | :--- |
| `env` | `dev`, `preprod`, `prod` | S3 Buckets, Secrets Manager secrets, Glue Jobs/Connections |
| `appname` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` | Data Lake S3 Buckets (Landing & Staging) |

### 6.2 Service Tag Condition Implementation Matrix

| Service | Environment | Applied Statement | Required `aws:ResourceTag/env` | Authorized `aws:ResourceTag/appname` |
| :--- | :--- | :--- | :--- | :--- |
| **AWS Glue** | `dev` | `S3DevDataLakeAccess`<br>`SecretsManagerAccess` | `dev` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **AWS Glue** | `preprod` | `S3PreprodDataLakeAccess`<br>`SecretsManagerAccess` | `preprod` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **AWS Glue** | `prod` | `S3ProdDataLakeAccess`<br>`SecretsManagerAccess` | `prod` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **MWAA** | `dev` | `S3DevDataLakeMonitorAccess` | `dev` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **MWAA** | `preprod` | `S3PreprodDataLakeMonitorAccess` | `preprod` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **MWAA** | `prod` | `S3ProdDataLakeMonitorAccess` | `prod` | `datalake-oepoeb`, `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **S3 Role** | `dev` | `S3DevStagingBucketReadAccess` | `dev` | `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **S3 Role** | `dev` | `S3DevLandingBucketIngestionAccess` | `dev` | `datalake-oepoeb`, `datalake-aratas` |
| **S3 Role** | `preprod` | `S3PreprodStagingBucketReadAccess` | `preprod` | `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **S3 Role** | `preprod` | `S3PreprodLandingBucketIngestionAccess` | `preprod` | `datalake-oepoeb`, `datalake-aratas` |
| **S3 Role** | `prod` | `S3ProdStagingBucketReadAccess` | `prod` | `datalake-oep`, `datalake-oeb`, `datalake-aratas` |
| **S3 Role** | `prod` | `S3ProdLandingBucketIngestionAccess` | `prod` | `datalake-oepoeb`, `datalake-aratas` |

### 6.3 Operational Rule: Platform DAGs & Scripts Exemption
> [!IMPORTANT]
> The Airflow DAGs/scripts bucket (`sst-s3-*-mwaa-dags`) is intentionally maintained in an independent IAM statement without the `aws:ResourceTag/appname` condition. Because this bucket stores orchestration scripts across multiple domain applications, applying an application-specific tag check to it would cause Glue job initializations and Airflow task runners to fail with `AccessDenied`.

