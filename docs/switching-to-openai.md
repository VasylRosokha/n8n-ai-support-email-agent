# Switching from Gemini to OpenAI

The workflows only talk to Gemini through n8n's model sub-nodes. Switching providers means swapping those nodes, rebuilding the Pinecone index and re-checking the threshold. Prompts, output schemas, routing and Gmail stay as they are.

Check the current OpenAI model names and embedding dimensions in your n8n version before you start; both change over time.

## 1. Add the OpenAI credential

In n8n: **+ (top of the left sidebar) → New credential → OpenAI**, paste the API key, save as `OpenAI`.

## 2. Swap the chat models (support agent)

| Node | Replace with |
|---|---|
| Gemini (classify) | OpenAI Chat Model, a small and fast model |
| Gemini (classify fallback) | OpenAI Chat Model, a different model |
| Gemini (answer) | OpenAI Chat Model, a larger model for better answers |
| Gemini (answer fallback) | OpenAI Chat Model, a different model |

Connect each new node to the same input of **Classify email** / **Write answer** (Model or Fallback Model). Keep the temperatures: 0 for classification, 0.2 for answers. The structured output parsers work with any chat model.

For the fallback, a model from a second provider (for example, keeping Gemini as the fallback) protects against one provider's outage. A different OpenAI model only protects against a single model being overloaded.

## 3. Swap the embeddings (both workflows)

Replace **Gemini embeddings** in **KB ingestion** and in **Support agent** with **Embeddings OpenAI**, using the same model in both places. Questions and knowledge base chunks must be embedded by the same model, or their scores mean nothing.

## 4. Create a new Pinecone index

Vectors from different embedding models can't be compared, even when they have the same length. Create a new index with the new model's dimension, and keep the old one so you can switch back:

```bash
read -rs PINECONE_API_KEY   # paste the key, press Enter
curl -s -X POST "https://api.pinecone.io/indexes" \
  -H "Content-Type: application/json" \
  -H "Api-Key: $PINECONE_API_KEY" \
  -H "X-Pinecone-Api-Version: 2026-07" \
  -d '{
    "name": "northwind-kb-openai",
    "deployment": {"deployment_type": "managed", "cloud": "aws", "region": "us-east-1"},
    "schema": {"fields": {"_values": {"type": "dense_vector", "dimension": 1536, "metric": "cosine"}}},
    "deletion_protection": "disabled"
  }'
unset PINECONE_API_KEY
```

Replace `1536` with your embedding model's output dimension. The index must be a *vector index* (a single `_values` field), the type n8n's Pinecone node works with; [setup.md](setup.md#43-create-the-index) explains why.

Then select `northwind-kb-openai` in **Insert into Pinecone** (KB ingestion) and **Search knowledge base** (Support agent).

## 5. Re-ingest the knowledge base

Open **Northwind – KB ingestion** and click **Execute workflow**. Pinecone should show 56 vectors in the namespace `kb`.

## 6. Re-tune the relevance threshold

Scores depend on the embedding model, so 0.62 won't carry over. Repeat the measurement from [testing.md](testing.md#relevance-threshold):

1. Send 5–6 questions the knowledge base answers and 5–6 it doesn't.
2. Read each run's top score from the alert email or the **Collect matches** node.
3. Set `relevance_threshold` in **Config** just below the lowest score of the questions it answers, as long as clearly unrelated questions stay below it.

The answer step still catches borderline questions, so the threshold only needs to filter out the obvious misses.

## 7. Re-test

Publish the support agent and re-run the scenarios in [testing.md](testing.md), at least: a covered question, a question not in the knowledge base, a discount request and two questions with only one covered.
