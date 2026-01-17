---
layout: default
title: Agentic DBT Generator
---

# Agentic DBT Generator

### A Multi-Agent System for generating and refining DBT projects and models

This system uses Langchain AWS module and orchestrator workflows with seperate agents for planning, generating and refining DBT projects and models.
Built with Infrastructure as Code (Terraform), this system demonstrates basic langchain patterns for AWS deployments on ECS & Fargate.

Review the README.md doc in github repo linked below for quickstart instructions

- [View Code on GitHub](https://github.com/cakeisgoodforyou/agentic-dbt-generator) 
- [See Architecture](agentic-dbt-generator-architecture)

---

## Key Features
- ✅ **Uses popular Langchain module for interacting with LLMs**
- ✅ **multi-Agent Architecture**: Planning, Generating and Refining agents work together in simple but powerful flow
- ✅ **Complete DBT Project Generation**: agents easily define complete projects and supply a build script to generate files locally
- ✅ **Fully Automated Deployment**: complete Terraform IaC
- ✅ **Code Versioning**: All LLM generated and refined code stored in S3 with timestamped run folders
- ✅ **Production Security**: Least privilege IAM, encrypted secrets, rate limiting

---

## Real-World Example

**Input:**
```
Prompt: "Generate staging models for customer data with incremental loading"
Source: customer_tbl in AWS Glue with 8 columns
```

**Output:**
A complete DBT project with SQL that:
- Pulls new customer records
- Updates changed customer records (address, phone, balance)
- Adds audit timestamps
- Uses DBT best practices
- Includes documentation

---

## How It Works (Simple View)

```
1. You Submit a Request
   ↓
2. System Reads Your Data Schema from AWS Glue
   ↓
3. AI Plans What to Build
   ↓
4. AI Writes the SQL Code
   ↓
5. AI Reviews and Improves the Code
   ↓
6. You Get a Complete DBT Project
```

## Use Cases

### 1. New Data Pipeline
You have 20 raw tables from a new data source. Generate staging models for all of them in one go.

### 2. Table Updates
A source table added 5 new columns. Regenerate the model to include them.

### 3. Best Practice Adoption
You have old models without proper incremental logic. Regenerate them with modern patterns.

### 4. Team Onboarding
New team members can see AI-generated examples of how to structure DBT code.

### 5. Rapid Prototyping
Quickly generate transformations to test different modeling approaches.

## Limitations

**What It's Good At:**
- Standard staging layer transformations
- Incremental loading patterns
- Common DBT patterns
- Consistent code structure

**What It's Not (Yet) Good At:**
- Complex business logic
- Multi-table joins with specific rules
- Domain-specific calculations
- Custom macros and packages

**Best Approach:**
Use it for the 80% of routine transformations, hand-code the 20% of complex logic.


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

---

## Clean Up

To avoid ongoing charges:

```bash
cd terraform

# Delete all resources
terraform destroy

# Confirm with 'yes'
```

---

[← Back to Main Documentation](index)