---
layout: single
title: "Multi-Agent RAG with S3 Vectors and Bedrock AgentCore"
date: 2026-02-01
categories: [blogging]
---

A RAG chatbot for personal notes using S3 Vectors and Bedrock Agents. Features true multi-agent collaboration with critique-driven feedback loop and multi-turn conversations. Built as a learning exercise after re:Invent 2025.

## Why I Built This

I already have [virtualme]([https://github.com/ox00004a/virtualme]) running in production. It's a RAG chatbot using DynamoDB for vector storage. It works, stays in free tier, does the job.

But re:Invent 2025 announced two things that caught my attention:

- **Amazon S3 Vectors** (December 2025). Native vector storage with server-side similarity search. Supports cosine, euclidean, and dot product metrics. Up to 10,000 dimensions per vector. Metadata filtering built-in. No more client-side cosine calculations or DynamoDB scan-and-compute patterns.

- **Amazon Bedrock AgentCore** (December 2025). Managed agent infrastructure with tool orchestration. Agents get knowledge base access, action groups, and session memory. Multi-agent setups possible: agents can invoke other agents, share context, or work in parallel. Built-in trace for debugging retrieval and reasoning steps.

I wanted to understand how these compare to my DynamoDB approach.

---

## Multi-Agent Architecture

True multi-agent collaboration with specialized roles:

- **Research Agent**: Searches knowledge base, extracts facts
- **Critique Agent**: Evaluates quality, provides feedback (1-10 scoring)
- **Formatter Agent**: Creates natural user responses

### Orchestration Pattern

Sequential agent handoff with feedback loop:

```
Query → Research Agent → Critique Agent → Score Check
                                      ↓
                              Score ≥ 7? → Formatter Agent → Response
                                      ↓
                              Score < 7? → Feedback to Research (max 3x)
```

### Session Management

- Shared session IDs across all agent calls
- Each query gets unique session ID
- All 3 agents share conversation memory
- Natural multi-turn conversations

---

## Architecture

```
                        DEPLOYMENT
                        ==========

./deploy.sh
     |
     v
+------------------+
|  S3 Bucket       |  <-- your .md/.txt notes go here
|  (notes/)        |
+--------+---------+
         |
         v
+------------------+
|  Bedrock         |  reads notes, chunks them
|  Knowledge Base  |  calls Titan Embeddings (1024 dims)
+--------+---------+
         |
         v
+------------------+
|  S3 Vectors      |  <-- vectors stored here
|  Index           |      native similarity search
+------------------+
```

```
                         QUERY FLOW
                         ==========

python client.py "What is DynamoDB?"
     |
     | HTTPS + SigV4 signing
     v
+------------------+
|  API Lambda      |
|  Function URL    |
+--------+---------+
         |
         v
+------------------+
|  Orchestrator    |  manages agent workflow
|  Lambda          |  extracts citations from trace
+--------+---------+
         |
         +---> Research Agent ---> Critique Agent
         |           ^                   |
         |           |     score < 7     |
         |           +-------------------+
         |                   |
         |            score >= 7
         |                   v
         +---> Formatter Agent ---> Response with Sources
```

### Orchestrator Lambda

The orchestrator is a Python Lambda (`orchestrator.handler`) that coordinates the multi-agent workflow. It receives the query, manages the critique loop, and assembles the final response.

Key implementation details:
- Calls `bedrock.invoke_agent()` with `enableTrace=True` to capture retrieval metadata
- Parses `knowledgeBaseLookupOutput.retrievedReferences` from trace to extract real S3 URIs
- Extracts filenames from URIs and appends them as sources (agents hallucinate filenames, so this is done server-side)
- Shares same `session_id` across all agent calls for conversation continuity
- Returns structured response with `iterations`, `final_score`, and `sources` for observability

---

## What I Wanted to Learn

### 1. AWS SAM

I've used Terraform before. Never SAM.

SAM handles Lambda packaging automatically. You point it at a directory, it zips and uploads:

```yaml
OrchestratorLambda:
  Type: AWS::Serverless::Function
  Properties:
    Runtime: python3.13
    Handler: orchestrator.handler
    CodeUri: ../lambda/    # SAM packages this
```

Run `sam build`, it creates `.aws-sam/build/` with deployment artifacts. Run `sam deploy --resolve-s3`, it handles the S3 bucket for you.

Good: Less config than raw CloudFormation.
Bad: Another abstraction layer to debug when things break.

### 2. S3 Vectors vs DynamoDB

**virtualme approach:**
```python
# Client-side similarity calculation
for item in dynamodb.scan():
    score = cosine_similarity(query_vector, item['embedding'])
```

**S3 Vectors approach:**
```yaml
VectorIndex:
  Type: AWS::S3Vectors::Index
  Properties:
    Dimension: 1024
    DistanceMetric: cosine   # server-side!
```

No Python similarity code. Bedrock handles the vector search natively.

Trade-off: S3 Vectors has a 2048-byte limit on filterable metadata per record. (see details below)

### 3. Multi-Agent Patterns

Understood the difference between:

- **Bedrock Flow**: Service orchestration, explicit control, no session memory between queries
- **Bedrock Agent**: Autonomous decision-making within a role, built-in session management
- **AgentCore**: Multi-agent collaboration, agents can delegate to each other

For this project, I chose individual agents coordinated by an orchestrator Lambda. This gives full control over the workflow while leveraging agent autonomy within each role.

---

## Key Technical Choices

1. **Agent Autonomy**: Each agent makes decisions within its role (search strategy, evaluation criteria, formatting style)
2. **Shared Sessions**: Context preservation across agent calls via session ID
3. **Feedback Loops**: Critique-driven improvement (max 3 iterations to limit cost)
4. **Source Extraction**: Orchestrator extracts real filenames from Bedrock trace (agents were hallucinating sources)
5. **Modular Architecture**: Infrastructure, agents, and API in separate nested stacks

---

## Things That Broke

### The 2048-byte Metadata Limit

First deployment. Ingestion job fails with `Filterable metadata must have at most 2048 bytes`.

Bedrock stores chunk text in filterable metadata by default. Even small paragraphs exceed 2048 bytes. I tried shorter filenames, smaller chunks, different chunking strategies. Nothing worked because the limit is per-record, not total.

The fix is to configure the index to treat Bedrock metadata as non-filterable:

```yaml
VectorIndex:
  Type: AWS::S3Vectors::Index
  Properties:
    MetadataConfiguration:
      NonFilterableMetadataKeys:
        - AMAZON_BEDROCK_TEXT
        - AMAZON_BEDROCK_METADATA
```

Catch: This must be set at index creation. I had to destroy the stack and redeploy.

### Agent Source Hallucination

Agents consistently invented plausible-sounding filenames instead of citing actual sources. Asked about SOLID principles, got "From Design Patterns Basics.md" when the real file was "SOLID & Design pattern.md".

Tried multiple approaches:
- Explicit instructions to copy exact filenames
- Critique agent penalizing invented sources
- Different prompt formats

None worked reliably. Nova Lite models don't seem to have clear visibility into the S3 URIs from retrieval results.

The fix to make it work: Extract citations in the orchestrator. Bedrock's `invoke_agent` response includes trace data with `knowledgeBaseLookupOutput.retrievedReferences`. Each reference has `location.s3Location.uri`. Parse the filename, append to response. Real sources, no hallucination.

---

## Cold Start Reality

First query after deployment or idle:

| Component | Time |
|-----------|------|
| API Lambda init | ~500ms |
| Orchestrator Lambda init | ~500ms |
| Bedrock Agent session | variable |
| Knowledge Base connection | first query slower |

**First query:** 15-30 seconds (everything cold)
**Warm queries:** 3-8 seconds
**After 15 min idle:** Agent session expires, partial cold start

virtualme has the same cold start issues. Serverless trade-off.

---

## Cost Comparison

### This Project (S3 Vectors + Multi-Agent)

| Component | Monthly Cost |
|-----------|-------------|
| S3 Vectors storage | < $0.01 |
| S3 Vectors queries | < $0.01 |
| Titan Embeddings (ingestion) | < $0.01 |
| Nova Lite (3 agents) | ~$0.30-0.70 |
| Lambda | free tier |
| **Total** | **~$0.35-0.75** |

Multi-agent multiplies Bedrock costs: 3 agents per query, up to 3 iterations if critique score < 7. Worst case: 9 model calls per query.

**Pricing reference** (Nova Lite): $0.06/1M input tokens, $0.24/1M output tokens.

### virtualme (DynamoDB)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| DynamoDB | $0 | Free tier: 25 RCUs/WCUs, 25GB |
| Nova Lite | ~$0.10-0.30 | Single agent per query |
| Lambda + API Gateway | $0 | Free tier: 1M requests/month |
| Route53 + CloudFront | ~$0.60 | Web frontend infrastructure |
| **Total** | **~$0.70-0.90** |

### Session Storage

Bedrock Agent sessions are configured with `IdleSessionTTLInSeconds: 600` (10 minutes). After 10 minutes of inactivity, the session expires and conversation context is lost.

Session storage itself is free. Idle time doesn't cost anything. What costs tokens is conversation history. The agent includes previous turns in each request:

```
Turn 1: "What is S3?"           →  ~50 input tokens
Turn 2: "Tell me more"          → ~150 input tokens (includes Turn 1)
Turn 3: "How about pricing?"    → ~300 input tokens (includes Turn 1+2)
```

For light usage (~100 queries/month), the difference is negligible.

### Final Outcome

DynamoDB is cheaper because I hacked it. In virtualme, I scan all items and calculate cosine similarity client-side. This works because my notes corpus produces fewer than 1000 chunks. Beyond that, it would get slow and expensive.

S3 Vectors costs slightly more but removes all that custom code. Native similarity search, no client-side calculations, no manual orchestration. For anything larger than a small personal project, managed infrastructure wins.

Multi-agent adds overhead. Each query involves 3+ model calls. Worth it for complex reasoning tasks, overkill for simple Q&A.

---

## Model Specialization (Future Improvement)

Currently all three agents use the same model (Nova 2 Lite). Using different models per role would improve results:

- **Different Strengths**: Each model brings different capabilities
- **Error Correction**: Claude might catch what Nova misses
- **Cost Optimization**: Use expensive models only where needed (e.g., Claude for critique, Nova for research)
- **Quality Improvement**: Specialized models for specialized tasks

I kept it simple for this learning exercise. Model specialization is a next step.

---

## Next Learning Path

### 1. Peer-to-Peer Agent Communication
Current architecture uses orchestrator-driven coordination. Agents don't talk to each other directly.

Next exploration:
- Research agent directly asks critique agent for guidance mid-search
- Agents negotiate who handles which part of a complex query
- Dynamic task decomposition without central orchestrator

This requires AgentCore's agent-to-agent invocation. Different trade-offs: more autonomous, but less predictable, should be harder to debug.

### 2. Model Specialization Experiment
- A/B test different model combinations per agent role
- Compare: Claude Haiku vs Nova Lite 2 vs Nova Micro 2
- Track: cost per query, response quality, iteration count

---

## What I Learned

1. **S3 Vectors has edge cases.** The 2048-byte metadata limit is not documented prominently. Cost me a few hours.

2. **Different LLMs behave differently.** Nova and Claude interpret the same agent prompts differently. Test with your actual model.

3. **Agents hallucinate sources.** Even with explicit instructions, models invent plausible filenames. Extract citations from the API trace, not the model output.

4. **SAM is convenient but adds abstraction.** When it works, great. When it breaks, you're debugging two layers.

5. **Multi-agent adds complexity and cost.** 3 agents × 3 iterations = 9 model calls worst case. Worth it for quality, overkill for simple queries.

6. **Orchestrator gives control.** Lambda-based orchestration lets you extract trace data, manage iterations, and append real sources. Pure agent-to-agent would lose this visibility.

7. **Free tier is hard to beat (in my case).** DynamoDB + custom code is cheaper only because I have fewer than 1000 chunks. This hack won't scale. For anything larger, managed services like S3 Vectors are the right choice.

---

## Resources

- [S3 Vectors announcement](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-vectors-preview/)
- [Bedrock AgentCore docs](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html)
- [virtualme (the DynamoDB approach)](https://github.com/jeremylem/virtualme)
