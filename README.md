# Query Open Pipeline for Crowdstrike Falcon Data Replicator

Query Open Pipeline for Crowdstrike Falcon Data Replicator (QOPCFDR) is an AWS native data mobility solution for Crowdstrike Falcon Data Replicator ETL into the Amazon Security Lake in OCSF v1.2.0 format.

QOPCFDR facilitates the mobility of streaming (and historical archives) of [Crowdstrike Falcon Data Replicator](https://www.crowdstrike.com/resources/data-sheets/falcon-data-replicator/) (FDR) data. FDR is a mechanism provided by Crowdstrike to: "Collect events in near real time from your Falcon endpoints and cloud workloads, identities and data, enriched by the AI-powered Crowdstrike Security Cloud."

FDR data provides incredibly rich and near real-time sensor-level data as well as other events such as interactions with the Crowdstrike API, identities, and other data points for use by incident response, threat hunting, and detection engineering teams. Crowdstrike provides [a Python script](https://github.com/CrowdStrike/FDR) that will poll your FDR-dedicated [Amazon Simple Queue Service (SQS)](https://aws.amazon.com/sqs/?nc2=h_ql_prod_ap_sqs) queue, download and parse objects containing FDR data from an Amazon S3 bucket, and write it to your own Amazon S3 bucket or local filesystem.

From that point forward is where QOPCFDR serves as an asset. Using Amazon Web Services (AWS) Cloud native services from analytics, application integration, serverless compute, and storage QOPCFDR handles batching, Extraction, Transformation, and Loading (ETL) of raw Crowdstrike FDR data into normalized and standardized [Open Cyber Security Format (OCSF)](https://github.com/ocsf/ocsf-docs/blob/main/Understanding%20OCSF.pdf?extensions=) [version 1.2.0](https://schema.ocsf.io/1.2.0/) and makes it available to the [Amazon Security Lake](https://aws.amazon.com/security-lake/).

As a community project we hope that current consumers of Crowdstrike FDR and/or the Amazon Security Lake find this solution beneficial. Additionally, given the wide breadth of FDR data that come from different operating systems, Crowdstrike licensing tiers, and capture operations - we only have a small snapshot (~120 events) of the nearly 1000 FDR events. We will accept pull requests to improve normalization, to expand mapped events, and share mappings.

## Solution Architecture

![QOPCFDR Solution Architecture](./media/QOPCFDR_Architecture.jpg)

From the bottom-left quadrant, the workflow is as follows:

1. Using the FDR Python script, FDR raw data is written into an Amazon S3 bucket.

2. An [Amazon EventBridge](https://aws.amazon.com/eventbridge/) Rule [monitors the S3 bucket](https://repost.aws/knowledge-center/eventbridge-rule-monitors-s3) for `ObjectCreation` events (Put, Copy, MultiPartUploadComplete) for FDR data being written (only `.gz` files).

3. EventBridge sends the objects to an SQS queue which batches the objects to an [AWS Lambda](https://aws.amazon.com/lambda/?nc2=h_ql_prod_fs_lbd) function.

4. The first Lambda function will parse and normalize the raw FDR data into JSON format, and send specific events to the appropriate upstream SQS queues based on the mapping of an FDR event to an [OCSF Class](https://schema.ocsf.io/1.2.0/classes?extensions=).

5. These subsequent SQS Queues batch up transformed data to Lambda functions which write into batches to dedicated [Amazon Data Firehose](https://aws.amazon.com/firehose/) Delivery Streams.

6. Data Firehose transforms the JSON data into [Parquet](https://parquet.apache.org/) format using schemas stored in [AWS Glue](https://aws.amazon.com/glue/) tables, and dynamically partitions the data in an appropriate format for Security Lake.

7. Each Firehose writes the GZIP-compressed Parquet data into [partitions](https://aws.amazon.com/blogs/big-data/get-started-managing-partitions-for-amazon-s3-tables-backed-by-the-aws-glue-data-catalog/) within a specific S3 location that matches an Amazon Security Lake Custom Source.

8. End-users can query the FDR Security Lake tables using [Amazon Athena](https://aws.amazon.com/athena/) - a [Trino](https://trino.io/)-based serverless analytics engine - or by using [Query.ai's Federated Search](https://www.query.ai/) platform.

9. Your analysts are very happy to use FDR data!

All services except for the Amazon Security Lake Custom Sources (and supporting ancillary services which are auto-created/auto-invoked) are deployed using AWS CloudFormation stacks. Refer to the [Prequisites & Assumptions](#prequisites--assumptions) and [Known limitations](#known-limitations) sections for information about what you need to do, and limitations around QOPCFDR, respectively.

## Prequisites & Assumptions

- You have at least Python 3.10.4 and `pip3` (or your favored package manager) installed locally.

- You have the AWS CLI configured on your local workstation, server, etc.

- Crowdstrike Falcon Data Replicator is enabled in your tenant.

- You use Crowdstrike's Python script for writing FDR data to Amazon S3.

- You have Security Lake enabled in at least one Account and Region.

- You have a separate security data collection account from your Security Lake Delegated Administrator, while this solution *can* work, additional Lake Formation permissions issues may arise that have not been tested..

- You have Admin access to your data collection and Security Lake account and can interact with various APIs from Lake Formation, Glue, S3, IAM, Firehose, SQS, and more.

- **YOU DO NOT CHANGE ANY HARD CODED NAMES!** To streamline deployment, a lot of names are hardcoded to avoid complex interpolation logic or repeated manual data entry (such as giving the wrong Firehose or SQS Queue name to the wrong Lambda function).

## Known Limitations

- This is a Security Lake-only integration, in the future we may expand to other Lakehouses using open source data orchestration projects.

- There is no utility to bulk-move previously saved FDR data. The best mechanism is to copy existing and future FDR dumps into a new bucket and key the automation off of it. In the future we are considering developing an EMR Notebook to utilize PySpark for petabyte scale mobility into the Security Lake.

- Only 122 out of 950+ Falcon Data Replicator event types are supported due to the size and scope of our environment. Nearly every Windows events and mobile events are missing. Advanced licensing data is missing from FDR as well. Please see the **Expanding Coverage** section for more information on contributing mappings or providing us data.

- Only the raw FDR events are normalized, other structured data from `aidmaster` and `manageddevice` is **NOT NORMALIZED NOR USED**.

- Not every potential normalization is known, and the normalization is written against OCSF v1.2.0.

- Simplistic exponential backoff built into Boto3 and Python is used for moving data between services - there can be times (depending on volume) where Firehose is throttled - Dead Letter Queues (DLQ) and more resilient retry logic will be developed at a later date.

## Deployment Steps

### Prepare the Security Lake Admin Account

The following steps must take place in the AWS Account where Amazon Security Lake is deployed. For the best configuration, deploy this into the (Delgated) Administrator account in your primary or "home" Roll-Up Region where all other Security Lake data flows.

1. Deploy the [`QOPCFDR_SchemaTransformation_CFN.yaml`](./src/cfn_yaml/QOPCFDR_SchemaTransformation_CFN.yaml) CloudFormation stack. This Stack deploys Glue Tables used by Firehose to translate JSON OCSF to Parquet, deploys necessary IAM Roles, and applies LakeFormation permissions to the Roles.

2. In the output of the Stack, copy all three of the ARNs for the Roles, as they will be needed in future steps as shown below. The immediate one that is needed is the Glue Crawler ARN. This has Lake Formation permissions and IAM permissions to crawl the FDR data written to the Security Lake S3 Bucket.

![Step 2](./media/step2.png)

3. Manually create Custom Sources using the Custom Source Name and Class Name detailed in [`qopcfdr_firehose_metadata.json`](./src/json/qopcfdr_firehose_metadata.json) as shown, and detailed, below. As of 7 JUNE 2024 you will need to manually create 13 Custom Sources.

    - Data source name: from `CustomSourceName`.
    - OCSF Event class: from `ClassName`. Note that `Operating System Patch State` appears as `Patch State` in Security Lake.
    - AWS Account ID: Enter your current account, this will create a dedicated IAM Role that allows s3:PutObject permissions. QOPCFDR uses Firehose so you can safely ignore these. *DO NOT DELETE THEM* or you will break Custom Sources.
    - External ID: Enter anything here.
    - Service Access: Select **Use an exisitng service role** and choose the Crawler Role deployed in Step 1. This will be named `query_open_pipeline_for_fdr_crawler_role`.

![Step 3](./media/step3.png)

4. Copy the the full S3 URI (including `s3://` and the trailing `/`) back into the JSON file as shown below. this file will be used by a script in the following steps that will create the Firehose Delivery Streams. This is also why we could not create the Firehose resources in the CloudFormation template.

![Step 4](./media/step4.png)

5. Run the `create_qopcfdr_firehoses.py` script, providing an argument of the Firehose Role name and optionally the location of the metadata JSON if it is not in your current working directory. This will create Firehose Delivery Streams that use the Glue tables and the IAM Role from the Stack deployed in Step 1.

- 5A: You will require the following IAM Permissions

```js
logs:CreateLogGroup
firehose:CreateDeliveryStream
firehose:ListDeliveryStreams
firehose:DeleteDeliveryStream
```

- 5B: Refer to the following commands to create a virtual environment, install necessary dependencies, and run the script.

```bash
# for macos
export AWS_ACCOUNT_ID="your_account_id"
brew install python3
pip3 install virtualenv
cd /Documents/GitHub/query_open_pipeline_for_crowdstrike_falcon_data_replicator
virtualenv -p python3 .
source ./bin/activate
pip3 install boto3 --no-cache
pip3 install argparse --no-cache
python3 ./src/python/create_qopcfdr_firehoses.py --firehose_role_arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/query_open_pipeline_for_fdr_firehose_role --firehose_metadata_file_name ./src/json/qopcfdr_firehose_metadata.json --deployment_mode create
```

To rollback the Firehose Delivery Streams, re-run the script without any arguments using `--deployment_mode delete`. It will keep the CloudWatch Log Group.

You have completed setup in the Security Lake Account, now change to your Data Collection AWS account where Crowdstrike FDR data is written to complete the setup.

### Prepare the the FDR Source Account

**IMPORTANT NOTE**: This solution has not been tested within one Account, proceed at your own risk if you attempt to deploy the rest of the solution into the (Delegated) Administrator Security Lake Account.

6. In the AWS Console navigate to **Services** -> **Storage** -> **S3**. In the S3 Console, create a bucket (or locate the existing one) for your Crowdstrike FDR data. In the **Properties** tab, scroll to the **Amazon EventBridge** section and enable the `Send notifications to Amazon EventBridge for all events in this bucket` option by using **Edit** as shown below.

![Step 6](./media/step6.png)

**NOTE**: It is recommended to create a new one if you are currently writing to a bucket that has `.gz` data written to it or uses a nested structure. This way you can safely copy existing FDR data and not risk transient failures from the QOPCFDR infrastructure being invoked for mismatched data.

- Identify the bucket where you will copy/write FDR day into. Ensure the setting `Send notifications to Amazon EventBridge for all events in this bucket` is enabled.

- Upload `QFDR_OCSF_Mapping.json`, `mapped_qfdr_events_to_class.json`, and `qopcfdr_stream_loader.py` into a (separate) S3 bucket.

- Deploy the `QOPCFDR_RealTimeCollection_CFN.yaml` Stack. This stack creates an EventBridge Rule, SQS Queues, Lambda functions, and IAM roles that facilitate the batching and queueing of FDR data from its raw form into the Security Lake.