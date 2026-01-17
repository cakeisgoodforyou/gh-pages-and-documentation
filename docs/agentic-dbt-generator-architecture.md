# Architecture

## System Overview

The Agentic DBT Generator is a serverless, event-driven system that uses multiple LLM agents to automatically generate production-quality DBT models from AWS Glue catalog schemas.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          User / CI/CD                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ aws ecs run-task
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS ECS (Fargate)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Orchestrator Container                       │  │
│  │                                                           │  │
│  │  ┌──────────────┐    ┌──────────────┐   ┌─────────────┐ │  │
│  │  │   Parsing    │───▶│   Planning   │──▶│ Generation  │ │  │
│  │  │   Workflow   │    │    Agent     │   │   Agent     │ │  │
│  │  └──────────────┘    └──────────────┘   └─────────────┘ │  │
│  │         │                    │                   │        │  │
│  │         │                    ▼                   ▼        │  │
│  │         │            ┌──────────────┐   ┌─────────────┐ │  │
│  │         │            │   Refining   │   │  S3 Logger  │ │  │
│  │         │            │    Agent     │   └─────────────┘ │  │
│  │         │            └──────────────┘           │        │  │
│  └─────────┼──────────────────┼──────────────────┼────────┘  │
└────────────┼──────────────────┼──────────────────┼───────────┘
             │                  │                  │
             ▼                  ▼                  ▼
    ┌────────────────┐  ┌─────────────┐  ┌──────────────────┐
    │  AWS Glue      │  │ AWS Bedrock │  │  S3 (Outputs)    │
    │  Data Catalog  │  │   Claude    │  │                  │
    │                │  │             │  │  runs/           │
    │  - Tables      │  └─────────────┘  │    run_*/        │
    │  - Schemas     │                   │      planning/   │
    │  - Metadata    │                   │      generation/ │
    └────────────────┘                   │      dbt_project/│
                                         └──────────────────┘
```

## Components

### 1. ECS Orchestrator (Fargate)

**Runtime:** Serverless container on AWS Fargate  
**Trigger:** Manual via AWS CLI or CI/CD pipeline  
**Purpose:** Execute the multi-agent workflow

**Environment Variables:**
- `PROMPT` - User's natural language request
- `SOURCE_DATABASE` - Glue database to analyze
- `SOURCE_TABLES` - Tables to process (with primary keys)
- `TARGET_DATABASE` - Where DBT models will target
- `NEW_PROJECT` - Create new or update existing
- `EXISTING_PROJECT_LOCATION` - S3 path to existing project (optional)

### 2. Multi-Agent Workflow

The system uses a three-agent pipeline:

#### **Planning Agent**
- **Input:** Glue schema + user prompt
- **Process:** Analyzes table structures, relationships, and requirements
- **Output:** Structured plan (JSON) defining models to create
- **Model:** Claude Haiku 4.5 (fast, cost-effective)

#### **Generation Agent**
- **Input:** Plan + Glue schema
- **Process:** Generates DBT SQL, YAML, and project structure
- **Output:** Complete DBT project files
- **Model:** Claude Haiku 4.5

#### **Refining Agent**
- **Input:** Generated DBT project + schema
- **Process:** Reviews and improves SQL quality, incremental logic
- **Output:** Refined DBT project ready for deployment
- **Model:** Claude Haiku 4.5

### 3. AWS Glue Data Catalog

**Purpose:** Source of truth for table schemas  
**Integration:** Boto3 SDK reads table metadata

**Retrieved Information:**
- Column names and types
- Table locations
- Partition information
- Comments/descriptions

### 4. AWS Bedrock (Claude)

**Model:** `global.anthropic.claude-haiku-4-5-20251001-v1:0`  
**Why Haiku:** Fast response times, cost-effective, sufficient for code generation

**Configuration:**
- Max tokens: 5000
- Temperature: 0.2 (focused, deterministic output)
- Region: us-east-1

### 5. S3 Storage

#### **Agent Logs Bucket**
- Path pattern: `runs/run_YYYYMMDD_HHMMSS/`
- Contents: JSON logs from each agent
- Lifecycle: Archive to IA after 30 days, Glacier after 90 days

#### **DBT Projects Bucket**
- Stores generated DBT projects
- Versioning enabled
- Source for `EXISTING_PROJECT_LOCATION`

## Data Flow

```
1. User submits task ──▶ ECS Task starts

2. Parse Schema:
   Glue Catalog ──▶ Table Metadata ──▶ Workflow

3. Planning:
   Schema + Prompt ──▶ Planning Agent (Bedrock) ──▶ Plan JSON

4. Generation:
   Plan + Schema ──▶ Generation Agent (Bedrock) ──▶ DBT Files

5. Refinement:
   DBT Files ──▶ Refining Agent (Bedrock) ──▶ Final DBT Project

6. Output:
   Final Project ──▶ S3 (runs/run_*/dbt_project/)
   All logs ──▶ S3 (runs/run_*/planning.json, etc.)
```

## Security

### IAM Permissions

**ECS Task Role:**
- `glue:GetTable` - Read Glue catalog
- `s3:GetObject`, `s3:PutObject` - Read/write S3
- `bedrock:InvokeModel` - Call LLM
- `logs:CreateLogStream`, `logs:PutLogEvents` - CloudWatch logging

### Network

- **ECS Tasks:** Run in VPC with public IP (for Bedrock API access)
- **Subnets:** Multi-AZ for availability
- **Security Groups:** Outbound HTTPS only

### Secrets

- No hardcoded credentials
- AWS IAM roles for all service access
- Environment variables for configuration

## Scalability

### Current Design
- **Concurrency:** 1 task per invocation
- **Throughput:** ~2-5 minutes per model generation
- **Cost:** Pay-per-use (ECS Fargate + Bedrock API calls)

### Future Scaling Options
- Run multiple ECS tasks in parallel
- Queue-based processing (SQS + Lambda trigger)
- Step Functions for complex workflows
- Caching of common schemas

## Technology Stack

**Infrastructure:**
- Terraform (IaC)
- AWS ECS Fargate (compute)
- AWS ECR (container registry)

**Application:**
- Python 3.11
- boto3 (AWS SDK)
- langchain-aws (Bedrock integration)
- langchain-core (agent framework)

**AI/ML:**
- AWS Bedrock
- Claude Haiku 4.5 (Anthropic)

**Data:**
- AWS Glue (catalog)
- AWS S3 (storage)
- AWS Athena (optional - for data preview)

## Cost Optimization

1. **Bedrock Model Choice:** Haiku is 100x cheaper than Opus
2. **Spot Instances:** Could use Fargate Spot for 70% savings
3. **S3 Lifecycle:** Auto-archive old logs
4. **Incremental Processing:** Only update changed models

## Monitoring

**CloudWatch Logs:**
- ECS task execution logs
- Agent conversation logs
- Error traces

**Metrics to Track:**
- Task success/failure rate
- Generation time per model
- Bedrock API latency
- Cost per generation

**Alerts:**
- Task failures
- Bedrock throttling
- S3 write errors