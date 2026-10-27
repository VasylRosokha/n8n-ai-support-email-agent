# Setup

How to get the support agent running on your own machine: n8n, Gmail, Gemini and Pinecone.

Tested with n8n 2.44.6. Screen and menu names in Google Cloud, Pinecone and n8n may differ slightly in other versions.

## What you need

- Docker with Docker Compose
- A dedicated Gmail account for the agent (the inbox it answers from)
- A second email account to send test emails from
- Free accounts at [Google AI Studio](https://aistudio.google.com) and [Pinecone](https://www.pinecone.io)
- An email address to receive escalation and error alerts, separate from the agent's inbox

## 1. Start n8n

```bash
cp .env.example .env
sed -i "s/^N8N_ENCRYPTION_KEY=.*/N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)/" .env
docker compose up -d
```

Open http://localhost:5678 and create the owner account. Keep a copy of `.env`: the encryption key protects the credentials n8n stores, and losing it means re-entering all of them.

## 2. Gmail

Do these steps signed in as the agent's Gmail account.

### 2.1 Google Cloud project

1. Open [Google Cloud Console](https://console.cloud.google.com) and create a project, e.g. `northwind-support-agent`.
2. **APIs & Services → Library**: find **Gmail API** and click **Enable**.
3. **Google Auth Platform** (opened from **APIs & Services → OAuth consent screen**) → **Get started**:
   - App name: `Northwind Support Agent`
   - User support email and contact email: the agent's Gmail address
   - Audience: **External**
4. **Audience → Test users → Add users**: add the agent's Gmail address and save. Leave the publishing status on **Testing**.
5. **Clients → Create client**:
   - Application type: **Web application**
   - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback` (n8n shows the same URL as **OAuth Redirect URL** in the credential form)
   - Copy the **Client ID** and **Client secret** right away; Google may not show the secret again.

### 2.2 n8n credential

1. In n8n: **+ (top of the left sidebar) → New credential → Gmail OAuth2 API**.
2. Paste the Client ID and Client secret, then click **Sign in with Google**.
3. Choose the agent account. Google warns that the app isn't verified: click **Continue** and allow all requested permissions.
4. Name the credential `Gmail – Northwind agent`.

**Sign-in expiry:** while the app's publishing status is **Testing**, Google ends the sign-in after 7 days. When Gmail nodes fail with an authorization error, open the credential and click **Sign in with Google** again. Do this before every demo. Publishing the app removes the limit, but Google may then require more details on the Branding page, such as a home page and privacy policy.

### 2.3 Gmail labels

In Gmail, create these five labels (left sidebar → **Labels → +**). The workflows use them to track each email's state:

| Label | Meaning |
|---|---|
| `ai-processed` | The agent has handled this email |
| `ai-replied` | The agent answered it |
| `not-support` | Not a support request, no reply |
| `needs-human` | Escalated to a person |
| `ai-skipped` | Filtered out before any AI call (newsletters, automated mail) |

## 3. Gemini

1. In [Google AI Studio](https://aistudio.google.com): **Get API key → Create API key**. You can attach it to the Google Cloud project from step 2.
2. In n8n: **+ (top of the left sidebar) → New credential → Google Gemini(PaLM) Api**. Leave **Host** as it is, paste the API key, save as `Gemini`.

The free tier is enough for this project but has per-minute rate limits; the workflow waits and retries when it hits them.

## 4. Pinecone

### 4.1 Check the embedding dimension

The Pinecone index must have the same dimension as the embedding model. The workflows use `gemini-embedding-001`, the default in n8n's Gemini embeddings node, which returns 3072 numbers per text. To confirm with your key:

```bash
read -rs GEMINI_KEY   # paste the key, press Enter
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-001:embedContent" \
  -H "x-goog-api-key: $GEMINI_KEY" -H "Content-Type: application/json" \
  -d '{"content":{"parts":[{"text":"test"}]}}' \
  | python3 -c "import sys, json; print(len(json.load(sys.stdin)['embedding']['values']))"
```

### 4.2 API key

In the [Pinecone console](https://app.pinecone.io): **API keys → Create API key**. Copy it; Pinecone shows it only once.

### 4.3 Create the index

n8n's Pinecone node reads and writes through Pinecone's Vectors API, which needs a **vector index**: one whose schema has a single dense field named `_values`. The console's **Create index** form doesn't accept that field name, so create the index with the API:

```bash
read -rs PINECONE_API_KEY   # paste the key, press Enter
curl -s -X POST "https://api.pinecone.io/indexes" \
  -H "Content-Type: application/json" \
  -H "Api-Key: $PINECONE_API_KEY" \
  -H "X-Pinecone-Api-Version: 2026-07" \
  -d '{
    "name": "northwind-kb",
    "deployment": {"deployment_type": "managed", "cloud": "aws", "region": "us-east-1"},
    "schema": {"fields": {"_values": {"type": "dense_vector", "dimension": 3072, "metric": "cosine"}}},
    "deletion_protection": "disabled"
  }'
unset PINECONE_API_KEY
```

The response should list `_values` with `"dimension":3072` and `"metric":"cosine"`. Use the dimension the check in 4.1 printed if it differs. The workflows write to the namespace `kb`; Pinecone creates it on first insert.

### 4.4 n8n credential

In n8n: **+ (top of the left sidebar) → New credential → Pinecone API**, paste the key, save as `Pinecone`.

## 5. Import and activate the workflows

1. In n8n: **Workflows → Create → Import from file**, once for each file in `workflows/`.
2. Open each imported workflow and select your credentials on any node that shows a warning.
3. In each workflow, open the **Config** node and set `agent_email` (the agent's Gmail address) and `alert_email` (where escalation and error alerts go).
4. Open **Northwind – KB ingestion** and run it once (**Execute workflow**). Pinecone should now show one vector per `##` section of the knowledge base.
5. In **Northwind – Support agent** and **Northwind – KB ingestion**: **Settings → Error workflow → Northwind – Error handler**.
6. Publish **Northwind – Error handler** (n8n only starts an error workflow that is published), then publish **Northwind – Support agent**.

Send a test email from your second account to the agent's address. Within a minute or two it should get a reply in the same thread and the label `ai-replied`.
