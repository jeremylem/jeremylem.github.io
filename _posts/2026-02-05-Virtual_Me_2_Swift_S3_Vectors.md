---
layout: single
title: "Virtual Me 2.0: 21x Faster Cold Starts with Swift and S3 Vectors"
date: 2026-02-05
categories: [blogging]
---

## The Evolution

A couple of weeks ago, I built [Virtual Me](https://chat.lemaire.tel),  a RAG-powered chatbot that answers questions about my professional experience. It worked, but I wasn't satisfied fully satisfied.

**The v1.0 stack:**
- Python Lambda with custom DynamoDB vector store
- LangGraph orchestration
- Manual embedding generation and chunking
- 4-second cold starts
- 1,488 lines of application code

It was over-engineered to avoid using ElasticSearch

So I rebuilt it from scratch. **Virtual Me 2.0** is now simpler, faster, and cheaper.

---

## What Changed

### Architecture Simplification

**Before (v1.0):**
```
Lambda (Python) → Custom Vector Search (DynamoDB) → Bedrock
```

**After (v2.0):**
```
Lambda (Swift) → Bedrock Knowledge Base → S3 Vectors → Bedrock
```

- **S3 Vectors** replaced my custom hacked DynamoDB similarity search
- **Bedrock Knowledge Base** handles document ingestion, chunking, and embedding automatically
- **Swift runtime** replaced Python for faster cold starts

### The Numbers

| Metric | v1.0 | v2.0 | Improvement |
|--------|------|------|-------------|
| **Cold Start** | ~4000ms | ~190ms | **21x faster** |
| **Application Code** | 1,488 lines | 205 lines | **86% reduction** |
| **Memory** | 512MB | 256MB | **50% reduction** |
| **Monthly Cost** | ~$3-5 | ~$2-3 | **40% cheaper** |

The cold start improvement is measured via CloudWatch Logs Insights:
- P50: 177ms
- P99: 214ms
- Range: 173-214ms

---

## Why Swift?

I first explored Swift server-side when I worked on a mobile app in my previous job. At the time, I experimented with Vapor and Kitura. I'm really happy to see the  language reached cloud.

I chose Swift for Lambda because of its compiled nature and minimal runtime overhead.

**Cold start breakdown:**
1. **Init Duration**: Time to initialize the Lambda execution environment
2. **Duration**: Actual function execution time

Python's interpreted nature means importing libraries (boto3, langchain, etc.) adds significant overhead to every cold start. Swift compiles to a native binary with all dependencies linked.

**The result is above what I was expecting:** 190ms cold starts vs 4000ms in Python.

### Swift Lambda Code

Here's the complete Lambda handler (simplified):

```swift
import AWSLambdaRuntime
import AWSLambdaEvents
import SotoBedrockAgentRuntime

@main
struct VirtualMeLambda {
    static func main() async throws {
        let runtime = LambdaRuntime { 
            (event: APIGatewayV2Request, context: LambdaContext) async throws -> APIGatewayV2Response in
            try await handleRequest(event: event)
        }
        try await runtime.run()
    }
}

func handleRequest(event: APIGatewayV2Request) async throws -> APIGatewayV2Response {
    guard let body = event.body else {
        return errorResponse(400, "Missing request body")
    }
    
    let request = try JSONDecoder().decode(ChatRequest.self, from: Data(body.utf8))
    let question = request.messages.last?.text ?? ""
    
    // Call Bedrock Knowledge Base
    let answer = try await retrieveAndGenerate(question)
    
    let responseBody = try JSONEncoder().encode(ChatResponse(text: answer))
    return APIGatewayV2Response(
        statusCode: .ok,
        headers: ["Content-Type": "application/json"],
        body: String(data: responseBody, encoding: .utf8)
    )
}
```

 No custom vector search. No manual embedding generation. Just call Bedrock Knowledge Base and return the result.

---

## S3 Vectors: Native Vector Storage

S3 Vectors is AWS's managed vector database. It integrates directly with Bedrock Knowledge Base.

**What it handles:**
- Vector indexing (automatic)
- Similarity search (sub-100ms)
- Scaling (automatic)
- Cost (pay per query, not per GB stored)

**What I don't have to build:**
- Cosine similarity calculations
- Vector normalization
- Index management
- Query optimization

My v1.0 DynamoDB implementation had ~400 lines of code for vector search. S3 Vectors replaced all of it. Not as free as a beer like DynamoDB but affordable for my budget. 

---

## Infrastructure: AWS SAM

I migrated from Terraform to AWS SAM (Serverless Application Model) for better Lambda development workflow.

**Why SAM:**
- `sam build` - Automatic dependency packaging
- Built-in best practices (IAM, X-Ray, CORS)
- Faster iteration cycle

Nested stacks keep the templates modular. Each stack is ~150 lines and specialized.

### Local Testing with Swift Lambda Runtime

The Swift AWS Lambda Runtime automatically starts a local HTTP server when not running in a Lambda execution environment. This makes testing incredibly simple:

```bash
# Start local server on http://127.0.0.1:7000/invoke
cd sam
make run-local
```

Under the hood, this runs:
```bash
KNOWLEDGE_BASE_ID=RPXCA7UUQN LLM_MODEL=nova-2-lite LLM_TEMPERATURE=0.1 swift run
```

**Test with curl:**
```bash
curl -X POST http://127.0.0.1:7000/invoke \
  -H "Content-Type: application/json" \
  -d '{
    "version": "2.0",
    "routeKey": "POST /chat",
    "body": "{\"messages\":[{\"role\":\"user\",\"text\":\"What is your experience?\"}]}"
  }'
```

This is much simpler than `sam local start-api` which requires Docker and emulates the entire API Gateway + Lambda stack. The Swift runtime's built-in local server connects directly to real AWS services (Bedrock, S3 Vectors) for authentic integration testing

---

## Measuring Cold Starts

CloudWatch automatically captures Lambda cold start metrics. I added a Makefile command to query them:

```bash
make cold-start-metrics
```

This runs a CloudWatch Logs Insights query:

```
fields @initDuration, @duration, @memorySize
| filter @type = "REPORT" and ispresent(@initDuration)
| stats avg(@initDuration) as avgColdStart,
        max(@initDuration) as maxColdStart,
        pct(@initDuration, 50) as p50ColdStart,
        pct(@initDuration, 99) as p99ColdStart,
        count() as totalColdStarts
```

The `@initDuration` field only appears on cold starts, making it easy to track.

---

## Lessons Learned

### 1. S3 Vectors 

My v1.0 custom DynamoDB vector store wasn't over-engineering, it was a necessary compromise. ElasticSearch was too expensive for a personal project. DynamoDB's free tier (25GB storage, 25 RCU/WCU) made it the only viable option for vector storage.

The complexity was the price of staying affordable.

Then S3 Vectors reached general availability on December 2, 2025, search became accessible for personal projects. The managed service eliminated 86% of my code while providing capabilities (sub-100ms similarity search, automatic indexing) that my DynamoDB implementation couldn't match.

**The lesson:** Sometimes complexity is justified by constraints. When those constraints change (new services, pricing models), it's worth revisiting your architecture.

### 2. Measure 

I assumed Python would be "fast enough" for cold starts. Measuring with CloudWatch proved otherwise. Swift's 21x improvement was worth the migration effort.

**Rule:** Measure before optimizing, but also measure to validate assumptions.

### 3. Compiled > Interpreted for Lambda

Python's flexibility comes at a cost: import overhead. Swift's compiled binary starts instantly.

For Lambda, prefer compiled languages over interpreted ones (Python, Node.js) when cold start matters.

### 4. Infrastructure as Code 

Terraform worked, but SAM's Lambda-specific features (automatic packaging) made development faster.

**Rule:** Choose IaC tools that match your workload. SAM for serverless, Terraform for multi-cloud.

---

## Cost Breakdown

**Monthly cost for ~100 conversations:**

| Service | v1.0 | v2.0 |
|---------|------|------|
| Lambda | $0.50 | $0.25 |
| DynamoDB | $1.50 | $0 |
| S3 Vectors | $0 | $0.75 |
| Bedrock | $1.50 | $1.00 |
| Other (S3, CloudFront, Route53) | $1.00 | $1.00 |
| **Total** | **~$4.50** | **~$3.00** |

The cost reduction comes from:
- 50% less Lambda memory (512MB → 256MB)
- No DynamoDB scan operations
- Bedrock Knowledge Base efficiency (fewer API calls)

---

## Try It Yourself

The complete source code is on GitHub: [github.com/jeremylem/virtualme2](https://github.com/jeremylem/virtualme2)

**Quick start:**
```bash
git clone https://github.com/jeremylem/virtualme2
cd virtualme2/sam
sam build
sam deploy --guided
```

**Local testing:**
```bash
make run-local          # Start Swift Lambda locally
make test-local         # Send test request
make cold-start-metrics # View CloudWatch metrics
```

Try the live version: [chat.lemaire.tel](https://chat.lemaire.tel)

---

**Tech Stack:** Swift 6.0 · AWS Lambda · Amazon Bedrock · S3 Vectors · AWS SAM · CloudFormation

**Performance:** 190ms cold starts · 256MB memory · $3/month

**Code:** 86% reduction · 62% total codebase reduction
