# Near Real-Time SQL to Apache Iceberg with AWS DMS and Amazon Data Firehose

This sample demonstrates a near real-time change data capture (CDC) pipeline that
replicates changes from a relational SQL database (for example, MySQL) into an
[Apache Iceberg](https://iceberg.apache.org/) table on Amazon S3, queryable through
Amazon Athena and the AWS Glue Data Catalog.

The pipeline is deployed as a single AWS CloudFormation template
([`sql-to-firehose-cfn.yaml`](sql-to-firehose-cfn.yaml)) with all IAM roles embedded,
so it is self-contained and easy to deploy for demonstration purposes.

> **Warning**
> This template creates billable AWS resources. Delete the stack when it is no
> longer needed to avoid ongoing charges. See [Cleanup](#cleanup).

## Architecture

```
Source SQL DB (CDC / binlog)
        │
        ▼
AWS DMS Serverless (CDC replication)
        │
        ▼
Amazon Kinesis Data Streams
        │
        ▼
Amazon Data Firehose  ──►  AWS Lambda (record transform / filter)
        │
        ▼
Apache Iceberg table on Amazon S3 (AWS Glue Data Catalog)
```

1. **AWS DMS Serverless** captures ongoing changes (CDC) from the source SQL
   database and writes them to an Amazon Kinesis Data Streams stream through a
   DMS Kinesis target endpoint.
2. **Amazon Kinesis Data Streams** buffers the change records.
3. **Amazon Data Firehose** reads from the stream and invokes an **AWS Lambda**
   function to transform and filter records, tagging each with the Iceberg
   destination database, table, and operation (insert/update/delete).
4. **Amazon Data Firehose** writes the processed records to an **Apache Iceberg**
   table on Amazon S3, performing upserts keyed on a configurable unique key.
   Failed records are written to an S3 error prefix (`firehose-errors/`).

### Resources created by the template

- Amazon Kinesis Data Streams stream (4 shards, 24h retention, KMS encrypted)
- AWS Lambda function and its Amazon CloudWatch Logs log group
- Amazon Data Firehose delivery stream (Iceberg destination) and log group
- AWS DMS Serverless replication configuration and Kinesis target endpoint
- AWS KMS key for CloudWatch Logs encryption
- AWS IAM roles (`lambda-role`, `firehose-role`, `dms-role`)

### Resources you provide (not created by the template)

- The source SQL database and its DMS source endpoint
- The Amazon S3 bucket and AWS Glue Iceberg table (Firehose destination)
- The DMS replication subnet group and VPC security group
- The AWS Secrets Manager secret used by the DMS source endpoint

## Prerequisites

Complete these before deploying. The template header contains links to the
relevant AWS documentation for each step.

1. **Source database** — a source database (for example, MySQL) with binlog
   enabled and a user that has replication privileges.
2. **DMS source endpoint** — an existing AWS DMS source endpoint pointing to your
   database. Note its ARN.
3. **AWS Secrets Manager secret** — the secret used by the DMS source endpoint.
   Note its ARN.
4. **Networking** — a DMS replication subnet group (≥2 AZs with connectivity to
   the source DB) and a security group allowing traffic to the source DB port.
   Note both IDs.
5. **Destination Iceberg table** — an Amazon S3 bucket and an AWS Glue Iceberg
   table for the Firehose destination.

You will also need:

- The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  installed and configured with credentials.
- Permission to create the resources listed above, including named IAM roles.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `DmsSubnetGroupId` | — | DMS replication subnet group ID |
| `DmsVpcSecurityGroupId` | — | DMS VPC security group ID |
| `DmsSourceEndpointArn` | — | ARN of the existing DMS source endpoint |
| `DmsSourceSecretArn` | — | ARN of the Secrets Manager secret used by the source endpoint |
| `DmsSourceTableName` | `market_rule` | Source table name to replicate |
| `GlueDatabaseName` | `market_db` | AWS Glue catalog database name |
| `GlueTableName` | `market_rule` | AWS Glue catalog table name |
| `S3BucketName` | `amzn-s3-demo-market-bucket` | S3 bucket backing the Iceberg table |
| `IcebergUniqueKey` | `rule_id` | Unique key column used for Iceberg upserts |

## Deployment

Deploy the stack with the AWS CLI (replace the parameter values with your own):

```bash
aws cloudformation create-stack --stack-name amzn-blog-demo \
  --template-body file://sql-to-firehose-cfn.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=DmsSourceEndpointArn,ParameterValue=arn:aws:dms:us-east-1:123456789012:endpoint:my-source-endpoint \
    ParameterKey=DmsSourceSecretArn,ParameterValue=arn:aws:secretsmanager:us-east-1:123456789012:secret:my-dms-secret \
    ParameterKey=DmsSubnetGroupId,ParameterValue=my-dms-subnet-group \
    ParameterKey=DmsVpcSecurityGroupId,ParameterValue=sg-0123456789abcdef0
```

## Testing

After the stack reaches `CREATE_COMPLETE`, verify the pipeline end to end:

```bash
# 1. Check stack status
aws cloudformation describe-stacks --stack-name amzn-blog-demo \
  --query 'Stacks[0].StackStatus'

# 2. Verify DMS replication is running (use the DmsReplicationConfigArn output)
aws dms describe-replications \
  --filters Name=replication-config-arn,Values=<DmsReplicationConfigArn>

# 3. Confirm the Kinesis stream is receiving records
aws kinesis describe-stream-summary --stream-name amzn-blog-demo-kinesis

# 4. Check Lambda invocations
aws cloudwatch get-metric-statistics --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=amzn-blog-demo-lambda-processor \
  --start-time <timestamp> --end-time <timestamp> \
  --period 300 --statistics Sum

# 5. Query the Iceberg table for replicated data
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM market_db.market_rule" \
  --result-configuration OutputLocation=s3://your-results-bucket/
```

Make an insert/update/delete on the source table and confirm the change appears
in the Iceberg table via Athena.

## Cleanup

To avoid ongoing charges, delete the stack when you are finished:

```bash
aws cloudformation delete-stack --stack-name amzn-blog-demo

# Verify deletion (should return DELETE_COMPLETE or "stack not found")
aws cloudformation describe-stacks --stack-name amzn-blog-demo
```

> **Note**
> The S3 bucket (`S3BucketName`) is **not** created or deleted by this template.
> After deleting the stack, the bucket and all Iceberg table data remain in your
> account and continue to incur storage charges. If you manually delete the
> bucket, all data is permanently lost — back up anything you need first.

The DMS source endpoint, DMS subnet group, and VPC security group are
pre-existing resources and are not deleted by the stack.

## Security considerations

This template is intended as a demonstration. Before adapting it for production,
consider the following (these are **not** configured by the template):

- The Lambda function's `LOG_LEVEL` defaults to `DEBUG`, which logs full record
  payloads that may contain sensitive business data. Set it to `INFO` or higher
  before deploying to production.
- IAM roles use inline policies for a self-contained sample; prefer managed
  policies and least-privilege scoping in production.
- Set up Amazon CloudWatch alarms for operational and security monitoring.
- Use VPC endpoints to keep traffic off the public internet.
- Encrypt inbound and outbound data stores.

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for how to report
a security issue.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
