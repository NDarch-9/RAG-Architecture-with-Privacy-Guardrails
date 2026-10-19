# Enterprise RAG-Architecture-with-Privacy-Guardrails
Cloud Infrastructure Design for an Internal AI Assistant: Document-Based Q&amp;A with Data Loss Prevention, Security, and Governance
## Problem Statement
Companies want to leverage Generative AI models (LLMs) to answer employee inquiries based on internal documents and policies
but they face major concerns regarding:


**1** - Leakage of sensitive data or Personally Identifiable Information (PII).

**2** - Prompt injection attacks.

**3** - Hallucinations and the lack of accurate sources for answers.

## **Architectural Solution:**
Deploying a secure cloud infrastructure based on RAG (Retrieval-Augmented Generation) technology using Amazon Bedrock and Knowledge Bases, featuring a rigorous security and protection layer via **Amazon Bedrock Guardrails**, along with end-to-end encryption for data at rest and in transit.
## Architecture Overview
<img width="2603" height="1772" alt="Blank diagram (5)" src="https://github.com/user-attachments/assets/9366d213-12ba-4e59-a8f2-2f059daa9387" />

## Security, Privacy & Observability

This design uses **defense in depth**: no single control protects the system on its own. Encryption, permissions, networking, and monitoring work together, so if one layer fails, the others still hold.

| Service | What it does here |
|---|---|
| **AWS KMS** (customer-managed keys) | Encrypts documents in S3 and vectors in OpenSearch. The company owns the keys and can revoke access at any time, which makes the data unreadable. |
| **AWS IAM** (least privilege) | Gives each component only the access it needs. For example, the Knowledge Base can only read documents (`s3:GetObject`) from one folder. |
| **VPC Endpoints** (PrivateLink) | Keeps traffic between services on AWS's private network, never on the public internet. |
| **Amazon CloudWatch** | Monitors the system and raises alerts on unusual activity, such as a sudden spike in blocked requests. |
| **AWS CloudTrail** | Records every API call: who did what, when, and from where. |

### Example: Why These Layers Matter

**Scenario:** A Lambda function's credentials leak. An attacker tries to read HR documents straight from S3, skipping Guardrails entirely.

**Expected behavior by design:**

1. **IAM** denies the request. The function's role has no permission on that folder.
2. **KMS** blocks decryption even if a permission was granted by mistake, because the key policy doesn't allow that role.
3. **VPC Endpoints** keep the services unreachable from the public internet.
4. **CloudTrail** logs the denied attempt with the identity, time, and source.
5. **CloudWatch** fires an alert after repeated denials, so the security team can react early.

**Takeaway:** Three layers prevent the attack (IAM, KMS, network) and two layers detect it (CloudTrail, CloudWatch). Nothing depends on a single control.

> **Note:** This scenario shows the expected behavior based on the architecture design. It has not been tested in a real AWS environment yet. This section will be updated with real results after implementation.
## Enterprise-RAG-Pipeline
<img width="2400" height="1480" alt="Enterprise-RAG-Pipeline" src="https://github.com/user-attachments/assets/5c2b6302-3203-4787-918a-e68c9afa0473" />


##  Service Justification

Each service below is listed in the order a request flows through the system, followed by the supporting services that protect and monitor it.

### Request path

| # | Service | Role | Why this choice |
|---|---|---|---|
| 1 | **Amazon API Gateway** | Entry point for employee questions | Managed front door with built-in authentication, throttling, and request validation. No servers to patch, and it stops abuse before it reaches the backend. |
| 2 | **AWS Lambda** | Orchestrator | Runs each step in order and reads the employee's identity from the verified token. Pay-per-request and scales automatically, which suits bursty internal usage. |
| 3 | **Amazon Bedrock Guardrails** (input) | Checks the question | Masks PII, blocks denied topics, and detects prompt injection before anything else runs. It is a managed policy layer, so safety rules do not depend on the model behaving well. |
| 4 | **Knowledge Bases for Amazon Bedrock** | Retrieval (the "R" in RAG) | Handles chunking, embedding, and retrieval as one managed workflow. This avoids building and maintaining a custom RAG pipeline. |
| 5 | **Amazon OpenSearch Serverless** | Vector index | Semantic search over document chunks with no cluster management. Supports metadata filters, which enable per-department access control. |
| 6 | **Amazon S3** | Source documents | Durable, cheap storage for company documents. Supports KMS encryption, versioning, and Block Public Access. |
| 7 | **Amazon Bedrock** (LLM) | Generates the answer | Access to foundation models through one API, with data kept inside the AWS account. The model answers only from retrieved context. |
| 8 | **Amazon Bedrock Guardrails** (output) | Checks the answer | Catches leaked data and inappropriate content, and verifies the answer is grounded in the retrieved sources, which reduces hallucinations. |

### Security and observability

| Service | Role | Why this choice |
|---|---|---|
| **AWS KMS** | Encryption keys | Customer-managed keys give the company control: it can rotate keys, audit their use, or revoke access. |
| **AWS IAM** | Permissions | Least privilege per component, so one compromised role cannot reach everything. |
| **VPC Endpoints (PrivateLink)** | Private networking | Service-to-service traffic never crosses the public internet. |
| **Amazon CloudWatch** | Monitoring and alerts | Detects unusual behavior early, such as a spike in blocked requests. |
| **AWS CloudTrail** | Audit trail | Records who called what, when, and from where, which supports governance and investigations. |


