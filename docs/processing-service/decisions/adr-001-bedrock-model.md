# ADR-001: AWS Bedrock for LLM Backend

## Status

Accepted

## Context

The processing service needs an LLM backend for claim extraction (Phase 1) and page synthesis (Phase 2). Options considered:

1. **AWS Bedrock** - Managed inference with IAM auth
2. **OpenAI API** - Direct API with API keys
3. **Self-hosted** - Run models on EC2/ECS
4. **Anthropic API** - Direct API with API keys

## Decision

Use **AWS Bedrock** as the LLM backend with **Qwen3-Coder 30b** as the default model.

## Rationale

### Why Bedrock

- **IAM authentication**: No API keys to manage, rotate, or secure
- **AWS integration**: Native VPC endpoints, CloudWatch metrics, X-Ray tracing
- **Multi-model**: Easy to switch models via configuration
- **SLA**: AWS enterprise support and availability guarantees
- **Cost**: Pay per token, no reserved capacity needed for variable workloads

### Why Qwen3-Coder 30b

- **Cost**: ~10x cheaper than Claude 3.5 Sonnet
- **Performance**: Optimized for code and structured extraction tasks
- **Context**: 32K context window sufficient for chunk-based processing
- **Quality**: Acceptable extraction quality for wiki synthesis

### Trade-offs Accepted

- **Vendor lock-in**: Bedrock-specific IAM and SDK patterns
- **Model limitations**: 32K context (vs 200K for Claude) requires chunking
- **Regional**: Must deploy in Bedrock-supported regions

## Alternatives Rejected

### OpenAI API

- Requires API key management
- No native AWS integration
- Higher latency from external calls

### Self-hosted

- Operational overhead for model hosting
- Capacity planning complexity
- Higher fixed costs

### Anthropic API

- Same issues as OpenAI
- Claude available through Bedrock anyway

## Consequences

- Lambda functions need `bedrock:InvokeModel` permission
- Rate limiting must account for Bedrock per-model limits
- Model switching is configuration-only (no code change)
- Must monitor Bedrock quotas and request increases proactively

## Related

- [Cost Model](../ops/cost-model.md) - Model pricing comparison
- [Feature 03](../features/03-phase1-claim-extraction.md) - Rate limiting implementation

## Date

2026-05-13
