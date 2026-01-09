---
layout: default
title: S3 Vectors URL Embed
---

# S3 Vectors URL Embed

### A Complete Infrastructure as Code Deployment for a RAG Vector Store Using AWS Bedrock and s3 Vectors

AWS Bedrock Knowledge Bases historically required OpenSearch for vector storage which can be costly and difficult to manage, especially for smaller projects and deployments which need to ship quickly.
By using the recently released (Dec 2024) S3 Vectors which offers pay-per-GB storage we can achieve a very large reduction in cost and also remove all the complexity of managing OpenSearch infrastructure and networking.

This project provides a complete terraform template to deploy everything you need for a Bedrock knowledge base backed by s3 Vectors.

- [View Code on GitHub](https://github.com/cakeisgoodforyou/s3-vectors-url-embed) 
- [See Architecture](s3-vectors-url-embed-architecture)

---

## Key Features
- ✅ **Cost Optimized**: 90% cheaper than OpenSearch  
- ✅ **Fully Automated Deployment**: Zero click-ops, all done by terraform for easy reproduction and rollback
- ✅ **Easy Ingest Public URLs**: Simply pass the URL you want to add to your vector store and chunking and embedding is handled
- ✅ **Handles Large Files**: Intelligent splitting with context preservation  
- ✅ **Production Security**: Least privilege IAM, encryption, no hardcoded secrets  
- ✅ **Clean Embeddings**: HTML → Plain text processing for meaningful results

---
## What Gets Deployed

***Note that costs are approximate and will vary depending on your usage.  Consult AWS pricing documentation if unsure and rollback all infra when finished with the project!***

| Component | Purpose | Cost/Month |
|-----------|---------|------------|
| Lambda Function | Fetch & process URLs | $0.10 |
| S3 Bucket (docs) | Store cleaned text | $0.05 |
| S3 Vectors | Store embeddings | $0.50-2 |
| Bedrock KB | RAG orchestration | $1.00 |
| **Total** | | **~$2/month** |

---

## Quick Start

```bash
# Clone and deploy
git clone https://github.com/cakeisgoodforyou/s3-vectors-url-embed.git
cd src/lambda/fetch_docs
pip install -r requirements.txt -t dependencies/
cd ../../../terraform
```

#### Edit Parameters for Terraform State File and Preferred Project Name
```
In backend.tf:

- bucket = "<my-s3-bucket-for-terraform-state-file>"
- key    = "<my-preferred-path>/terraform.tfstate"
- dynamodb_table = "<my-dynamoDB-table-for-storing-state-file-lock-info>"
```

#### Edit the terraform project_name variable to your preferred project name
```
In Locals.tf:

- project_name = "<my-preffered-project-name>"
- environment  = var.environment #(or set to dev, test etc)
```

#### Initialise the terraform environment and deploy resources
```bash
terraform init
terraform apply
```

---

## Post Deployment Testing

Invoke the fetch_docs lambda function to parse a public URL:

```bash
PAYLOAD=$(echo '{"urls": ["https://docs.stripe.com/api/authentication"], "prefix": "stripe-api-docs"}' | base64)
aws lambda invoke \
  --function-name <my-project-name>-<my-env>-fetch-docs \
  --payload "$PAYLOAD" \
  --region us-east-1 \
  response.json
```

### Sync the knowledge base in Bedrock Console (or via API call)

The easiest way to complete ingestion is to navigate to your new knowledge base in bedrock console, select the new datasource we created and hit the sync button.
Alternatively you can gather the required IDs using the AWS CLI and start the sync progratically like below.
```bash
aws bedrock-agent start-ingestion-job \
    --knowledge-base-id <your-knowledge-base-id> \
    --data-source-id <your-data-source-id> \
    --region <your-region-name>
```

---

## Important Callouts ##

### Non-Filterable Metadata

By default AWS passes the entire contents of the txt document derived from your chosen URL to the vector's metadata object "AMAZON_BEDROCK_TEXT".  

***This will cause bedrock ingestion to fail due to exceeding a 2KB limit for metadata.***. To avoid these we need to set AMAZON_BEDROCK_TEXT as non filterable metadata.

As of hashicorp/aws terraform module version 6.27.0 there is no configuration option for nonFilterableMetadataKeys in terraform directly so we instead deploy the required s3 Vector Index and configuration via a null resource.

Critical configuration for S3 Vectors:

```terraform
resource "null_resource" "create_vector_index" {
  depends_on = [aws_s3vectors_vector_bucket.s3_vectors_url_embed_vector_store]

  provisioner "local-exec" {
    command = <<EOT
      aws s3vectors create-index \
        --index-name ${local.name_prefix}-vectors-index \
        --vector-bucket-name ${aws_s3vectors_vector_bucket.s3_vectors_url_embed_vector_store.vector_bucket_name} \
        --dimension 256 \
        --distance-metric euclidean \
        --data-type float32 \
        --metadata-configuration '{"nonFilterableMetadataKeys":["AMAZON_BEDROCK_METADATA","AMAZON_BEDROCK_TEXT","x-amz-bedrock-kb-data-source-id","S3VECTORS-EMBED-SRC-LOCATION","S3VECTORS-EMBED-SRC-CONTENT"]}'
    EOT
  }
}
```
Without this, large metadata exceeds the 2KB limit and ingestion syncs will fail in many cases.

**The solution above resolves the issue but if you are running terraform destroy or rolling back / changing resources you may need to delete the vector bucket's index manually**
