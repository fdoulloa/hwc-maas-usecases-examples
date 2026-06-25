# 🤖 Support Ticket Assistant

A Dify workflow triggered by a webhook that automatically classifies incoming support tickets, retrieves relevant knowledge base context, and either escalates to a human agent via email or sends a Slack notification — powered by DeepSeek on Huawei Cloud MaaS.

## Overview

```
[Webhook] → [Code: Parse Ticket] → [Parameter Extractor] → [Question Classifier]
                                                                  ├── [Class 1] → [Knowledge Retrieval] → [LLM L1 Support] → [send email]
                                                                  └── [Class 2] → [LLM Escalation] → [Slack notification]
                                                                  └── [Default]  → [End]
```

## Flow

| Step | Node | Description |
|------|------|-------------|
| 1 | **Webhook Trigger** | Receives ticket payload via HTTP POST (async mode) |
| 2 | **Code: Parse Ticket** | Extracts fields from raw webhook body: `ticket_number`, `title`, `body`, `priority`, `submitter`, `status`, `created_at`, `submitter_email` |
| 3 | **Parameter Extractor** | Structures ticket data for downstream nodes |
| 4 | **Question Classifier** | Routes ticket based on content: Class 1 (knowledge base query) or Class 2 (escalation) |
| 5a | **Knowledge Retrieval** | Searches Huawei Cloud knowledge base using ticket description |
| 5b | **LLM – L1 Support Assistant** | Generates a suggested resolution using KB context (DeepSeek V3.2 via Huawei MaaS, temp=0) |
| 5c | **send email** | Sends resolution suggestion to `sebastian.george1@huawei.com` with subject `Requerimiento de Producto #<ticket_number>` |
| 6 | **LLM – Escalation** | Generates escalation message for Class 2 tickets |
| 7 | **Slack notification** | Posts escalation to a Slack channel |
| 8 | **End** | Outputs ticket description for unclassified tickets |

## Webhook Payload Format

```json
{
  "ticket_number": "TKT-001",
  "title": "Cannot access the portal",
  "body": "User reports login page returns 500 error since yesterday.",
  "priority": "high",
  "submitter": "John Doe",
  "status": "open",
  "created_at": "2025-01-15T10:30:00Z",
  "submitter_email": "john.doe@example.com"
}
```

## Dependencies

| Plugin | Version |
|--------|---------|
| `langgenius/email` | `0.0.16` |
| `langgenius/maas` | `0.0.21` |
| `langgenius/slack` | `0.0.9` |

## Models Used

| Node | Model | Provider |
|------|-------|----------|
| L1 Support Assistant | `deepseek-v3.2` | Huawei Cloud MaaS |
| Escalation LLM | `deepseek-v3.2` | Huawei Cloud MaaS |
| Knowledge Reranker | `bge-reranker-v2-m3` | Huawei Cloud MaaS |
| Embeddings | `qwen3-embedding:8b` | Ollama |

## Knowledge Base

The workflow uses a vector knowledge base configured with:
- **Retrieval mode:** Multiple (vector + keyword)
- **Top K:** 4
- **Reranking:** enabled (`bge-reranker-v2-m3`)

## Environment Variables

| Variable | Type | Description |
|----------|------|-------------|
| `SKIP_TRIGGER_URL` | string | Debug webhook URL (used during testing) |

## Configuration Checklist

Before importing this workflow:

- [ ] Replace the email `send_to` value with your target support email
- [ ] Connect your Slack workspace via the Slack plugin settings
- [ ] Link your knowledge base dataset (replace the `dataset_ids` reference)
- [ ] Configure Huawei Cloud MaaS credentials in the MaaS plugin
- [ ] Update `SKIP_TRIGGER_URL` with your own debug webhook endpoint

## Import

1. In Dify, go to **Studio → Import DSL**
2. Upload `Support_Ticket_Assistant.yml`
3. Complete the configuration checklist above
4. Trigger the webhook with a sample ticket payload to test the flow
