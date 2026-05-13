# Cost Model

> Cost estimates for capacity planning. Actual costs vary by usage patterns.

---

## Precision Caveat

The estimates below are **order-of-magnitude guidance**, not budgeting targets. Actual costs depend on:

- PDF content complexity
- Chunk counts per PDF
- LLM output verbosity
- Claim density
- Page synthesis complexity

Use these numbers for **capacity planning**, not invoicing.

---

## Bedrock Model Pricing

Default model: **Qwen3-Coder 30b** (`qwen.qwen3-coder-30b-a3b-v1:0`)

| Model                         | Input (per 1K tokens) | Output (per 1K tokens) | Notes                       |
| ----------------------------- | --------------------- | ---------------------- | --------------------------- |
| **qwen3-coder-30b** (default) | ~$0.00035             | ~$0.0014               | ~10x cheaper than Claude    |
| claude-3-5-sonnet             | ~$0.003               | ~$0.015                | Higher context, higher cost |
| claude-3-haiku                | ~$0.00025             | ~$0.00125              | Fast, cheap, less capable   |

> **Note**: Verify current pricing in AWS documentation. Prices change over time.

---

## Per-PDF Cost Breakdown (Qwen3-Coder 30b)

Typical 50-page PDF:

| Component            | Estimated Usage   | Estimated Cost |
| -------------------- | ----------------- | -------------- |
| Lambda (Phase 0)     | ~15 GB-s          | ~$0.00025      |
| Lambda (Phase 1-2)   | ~100 GB-s         | ~$0.002        |
| Bedrock (extraction) | ~50K in, ~15K out | ~$0.04         |
| Bedrock (synthesis)  | ~20K in, ~20K out | ~$0.04         |
| S3 storage           | ~5 MB             | ~$0.0001       |
| DynamoDB             | ~50 writes        | ~$0.0001       |
| **Total**            |                   | **~$0.08**     |

---

## Batch Cost Estimates (60 PDFs)

| PDF Size Category        | Cost (Qwen) | Cost (Claude) | Notes                  |
| ------------------------ | ----------- | ------------- | ---------------------- |
| 60 small (< 50 pages)    | ~$5         | ~$30          | Mostly LLM costs       |
| 60 medium (50-200 pages) | ~$10        | ~$60          | More extraction chunks |
| 60 large (200+ pages)    | ~$20        | ~$120         | ECS for Phase 0        |

---

## Monthly Steady-State Estimates

| Usage Level     | Cost (Qwen) | Cost (Claude) |
| --------------- | ----------- | ------------- |
| 100 PDFs/month  | ~$10        | ~$60          |
| 500 PDFs/month  | ~$50        | ~$300         |
| 1000 PDFs/month | ~$100       | ~$600         |

---

## Infrastructure Fixed Costs

| Resource                | Monthly Cost | Notes                       |
| ----------------------- | ------------ | --------------------------- |
| DynamoDB (on-demand)    | $0           | Pay per request             |
| S3 storage              | ~$0.02/GB    | Wiki + processing artifacts |
| CloudWatch Logs         | ~$0.50/GB    | Ingestion + storage         |
| Secrets Manager         | ~$0.40       | Per secret per month        |
| **Baseline (no usage)** | ~$5/month    | Minimal fixed costs         |

---

## Cost Optimization Strategies

### Model Selection

| Strategy                   | Savings | Trade-off                |
| -------------------------- | ------- | ------------------------ |
| Use Qwen instead of Claude | ~10x    | Slightly less capable    |
| Use Haiku for simple files | ~2x     | Less reliable extraction |

### Concurrency Tuning

| Strategy                | Effect                              |
| ----------------------- | ----------------------------------- |
| Lower Phase1Concurrency | Slower but fewer rate limit retries |
| Lower Phase2Concurrency | Slower but fewer rate limit retries |

### Lifecycle Optimization

| Strategy                   | Savings          |
| -------------------------- | ---------------- |
| Shorter artifact retention | S3 storage costs |
| Log retention reduction    | CloudWatch costs |

---

## Cost Monitoring

### CloudWatch Metrics

| Metric                | Dimension | Purpose                |
| --------------------- | --------- | ---------------------- |
| Bedrock input tokens  | model     | Track LLM input costs  |
| Bedrock output tokens | model     | Track LLM output costs |
| Lambda duration       | function  | Track compute costs    |
| S3 requests           | bucket    | Track request costs    |

### Recommended Alarms

| Alarm                   | Threshold       | Action                  |
| ----------------------- | --------------- | ----------------------- |
| Daily Bedrock cost      | > $50           | Alert for investigation |
| Monthly cost projection | > budget \* 0.8 | Budget warning          |

---

## Pricing Verification

Before production deployment, verify pricing:

1. Check AWS Bedrock pricing page for current model costs
2. Check Lambda pricing for memory/duration costs
3. Check S3 pricing for storage and request costs
4. Run cost projection with expected volume

---

## Changelog

| Date       | Change                       |
| ---------- | ---------------------------- |
| 2026-05-13 | Initial cost model extracted |
