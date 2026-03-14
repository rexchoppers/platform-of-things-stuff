---
layout: post
title:  "Trakkt - Dev Blog #1 - RAG"
category: Projects
---

# Overview
Trakkt is a project I've been working on in the background for a while now. Whilst it's under wraps on what it does currently, I can share some unique insights into the development process, and the technologies I'm using to build it.

The first major hurdle I have is to outline the templates for a machine. Excavators, bulldozers, cranes, earth movers, and more. Each machine has its own unique set of data points, service intervals, maintenance requirements, and more. I have a lot (and I mean a lot) of unstructured, and public data on these machines (hundreds of megabytes for each machine) that I need to structure in a standard format, so I can use it to power the app. Welcome to the world of RAG (Retrieval Augmented Generation).

# 1 - Prototype
The initial prototype of the templating system involved locally ingesting the PDF data into QDrant (Using OCR where necessary), and then using a Phi model to query the data, and generate a structured JSON output. This actually worked quite well: Templates generated, mostly correct data and a good starting point for the next phase of development.

Problem? Well for my electricity supplier, it was like all their Christmases came at once. I had to generate templates for hundreds of machines, and each machine had hundreds of megabytes of data. Too slow, and too expensive to do locally. Onwards to the cloud.

# 2 - Digital Ocean
I love DigitalOcean when a quick solution is required and cuts the need for a lot of DevOps work. They have a pretty robust Knowledge Base system via their Gradient AI solution. This spins up a OpenSearch cluster for ingestion and provides a serverless inference API for the model. 

Pretty cool - but the deal breaker? Not being able to filter by metadata. Machine data may have duplicate data points across different machines, and I need to be able to filter by machine type, model, and more.

# 3 - AWS Bedrock
AWS Bedrock to the rescue. Bedrock provided the following: 

- A piss easy way to ingest data. S3 buckets, setup S3 Vectors (No OpenSearch/Chroma/QDrant management)
- A serverless inference API for the model - The Amazon Titan family of models is pretty good, and the Bedrock API is super easy to use.
- Metadata filtering - I could filter by machine type and model which reduces the amount of data for the model to process, and increases the accuracy of the output.

Wee bit more expensive but at the minute, I needed quick, reliabile and something that will scale. There are plenty more solutions to explore in the future, but for hitting the ground running, Bedrock was the way to go.

# 4 - Process Pipeline
- Data Sync
    - Sync raw data to S3 bucket w/ metadata file (Machine type, model, data type, etc)
    - Trigger Knowledge Base sync on new data
- Generate JSON Template
    - Call model via Bedrock API with the prompts below:
        - Prompt 1
            - Summarise data into a structured format. Provided a list of data points to extract, and the model summarises the data into a list format
        - Prompt 2
            - Take the structured list output from Prompt 1, and generate a JSON template for the machine. This template will be used in the app development to power the machine profiles, service reminders, and more.
            - A second prompt is required to convert the structured list into a JSON format, as the model struggles to output in a consistent JSON format in one go.
- Final JSON Polish
    - A final prompt to polish the JSON output, and ensure it's in a consistent format for use in the app development. This is where the human in the loop comes in for quality control, and to fix any issues with the output.
- Publish Template
    - Output JSON template to S3 bucket for use in app development. This will be used to power the machine profiles

Done - In a matter of hours, hundreds of machines can have templates generated, with minimal manual intervention. There is a human at the end reviewing the output for quality control, but the time saved is immense.

# 5 - CDK Defnitions

**Note**: Some parts have been modified for privacy reasons, but the general structure is the same. Also, this is a very rough working example - expand upon it as you see fit.

```python

# Separate construct until CDK supports this natively. 
# https://github.com/bimnett/cdk-s3-vectors
import cdk_s3_vectors as s3_vectors

class TrakktTemplatesStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # -------------------------
        # S3 — sources bucket (documents)
        # -------------------------
        self.bucket = s3.Bucket(
            self,
            "SourcesBucket",
            bucket_name=f"sources-{Aws.ACCOUNT_ID}",
            versioned=True,
            encryption=s3.BucketEncryption.S3_MANAGED,
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL,
            removal_policy=RemovalPolicy.RETAIN,
        )

        # -------------------------
        # S3 Vector Bucket + Index
        # -------------------------
        self.vector_bucket = s3_vectors.Bucket(
            self,
            "VectorBucket",
            vector_bucket_name="sources",
        )

        self.vector_index = s3_vectors.Index(
            self,
            "VectorIndex",
            index_name="sources",
            vector_bucket_name=self.vector_bucket.vector_bucket_name,
            dimension=1024,
            distance_metric="cosine",
            data_type="float32",
            metadata_configuration={
                "non_filterable_metadata_keys": [
                    "AMAZON_BEDROCK_TEXT",
                    "AMAZON_BEDROCK_METADATA",
                    "AMAZON_BEDROCK_TEXT_CHUNK",
                ],
            },
        )
        self.vector_index.node.add_dependency(self.vector_bucket)

        # -------------------------
        # Bedrock Knowledge Base
        # -------------------------
        self.knowledge_base = s3_vectors.KnowledgeBase(
            self,
            "KnowledgeBase",
            knowledge_base_name="trakkt-templates",
            vector_bucket_arn=self.vector_bucket.vector_bucket_arn,
            index_arn=self.vector_index.index_arn,
            knowledge_base_configuration={
                "embedding_model_arn": f"arn:aws:bedrock:{Aws.REGION}::foundation-model/amazon.titan-embed-text-v2:0",
                "dimensions": "1024",
            },
        )
        self.knowledge_base.node.add_dependency(self.vector_index)
        self.knowledge_base.node.add_dependency(self.vector_bucket)

        # -------------------------
        # Data source (S3 → KB)
        # -------------------------
        self.data_source = bedrock.CfnDataSource(
            self,
            "DataSource",
            name="sources",
            knowledge_base_id=self.knowledge_base.knowledge_base_id,
            data_source_configuration={
                "type": "S3",
                "s3Configuration": {
                    "bucketArn": self.bucket.bucket_arn,
                },
            },
        )
        self.data_source.node.add_dependency(self.knowledge_base)
        self.data_source.node.add_dependency(self.bucket)

        # Allow KB role to read source documents
        self.bucket.grant_read(self.knowledge_base.role)

        # -------------------------
        # IAM — pipeline role
        # -------------------------
        self.pipeline_role = iam.Role(
            self,
            "PipelineRole",
            role_name="trakkt-templates-pipeline",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
        )

        # S3 access
        self.bucket.grant_read_write(self.pipeline_role)

        # Bedrock — model invocation
        self.pipeline_role.add_to_policy(
            iam.PolicyStatement(
                actions=[
                    "bedrock:InvokeModel",
                    "bedrock:InvokeModelWithResponseStream",
                ],
                resources=[
                    f"arn:aws:bedrock:{Aws.REGION}::foundation-model/amazon.nova-lite-v1:0",
                    f"arn:aws:bedrock:{Aws.REGION}::foundation-model/amazon.titan-embed-text-v2:0",
                ],
            )
        )

        # KB retrieval + ingestion
        self.knowledge_base.grant_ingestion(self.pipeline_role)
        self.pipeline_role.add_to_policy(
            iam.PolicyStatement(
                actions=["bedrock:Retrieve"],
                resources=[self.knowledge_base.knowledge_base_arn],
            )
        )

```

# 6 - Final Output
MBs of unstructured data data converts to: 

```json
{
  "identifier": "templates/assets/mobile-plant/liebherr-a-918-compact-litronic-gen-8",
  "reference": "templates/assets/base/mobile-plant",
  "name": "Liebherr A 918 Compact Litronic (Gen 8)",
  "key": "liebherr-a-918-compact-litronic-gen-8",
  "fields": {
    "generation": 8,
    "model": "A 918 Compact Litronic (Gen 8)",
    "manufacturer": "liebherr",
    "type": "wheeled-excavator",
    "engines": [
      {
        "manufacturer": "fpt",
        "model": "D924",
        "serial": null,
        "fuelType": "diesel",
        "cylinderCount": 4,
        "cylinderArrangement": "in-line"
      }
    ],
    "vin": null,
    "manufactureDate": null,
    "manufactureCountry": null
  }
}
```

# Final Thoughts
This is what AI should be doing. Accelerating the boring, time consuming, and repetitive tasks; providing real value for businesses and individuals alike.

Basically anything than using half the ocean to generating a dog dancing video.