# Store Policies Upload

A small n8n workflow that gives you an upload form for store documents (return policy,
shipping info, FAQ, About us…). Uploaded files are chunked, embedded with OpenAI
`text-embedding-3-small` and stored in a Qdrant collection that the WhatsApp Sales Agent
searches when customers ask questions about the store.

**Workflow file:** `Store Policies Upload.json`

## What it does

1. Serves a password-protected **upload form** (title, category, file —
   `.pdf`, `.docx`, `.txt`, `.md`, `.csv`).
2. Ensures a keyword index exists on `metadata.doc_title` in Qdrant (needed for step 3).
3. Deletes any chunks previously stored under the same document title, so re-uploading a
   document **replaces** it instead of duplicating it.
4. Splits the file into chunks (recursive splitter, 200-char overlap), embeds them and
   inserts them into the `woocommerce-policies` collection with metadata
   `doc_title`, `category`, `uploaded_at`.
5. Returns a summary: document name, category, number of chunks stored.

## Prerequisites / subscriptions

| Service | What you need | Cost |
|---|---|---|
| n8n | A running instance with the LangChain nodes | — |
| [Qdrant Cloud](https://cloud.qdrant.io/) | Free tier cluster (can be the same cluster as the products sync) | Free |
| [OpenAI](https://platform.openai.com/) | API key for `text-embedding-3-small` embeddings | Pay-per-use |

## 1. Create the Qdrant collection

The policies collection is **`woocommerce-policies`**. Like the products collection,
it must use vector size **1536** and **Cosine** distance to match
`text-embedding-3-small`:

```bash
curl -X PUT "https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-policies" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 1536,
      "distance": "Cosine"
    }
  }'
```

You do **not** need to create the payload index by hand — the workflow's
**Ensure Doc-Title Index** node creates the keyword index on `metadata.doc_title`
automatically on each run (it's idempotent). If you prefer to create it manually:

```bash
curl -X PUT "https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-policies/index?wait=true" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "field_name": "metadata.doc_title", "field_schema": "keyword" }'
```

Reference: [Qdrant payload indexing docs](https://qdrant.tech/documentation/concepts/indexing/).

## 2. Configure the workflow after import

1. **Import** `Store Policies Upload.json` into n8n (*Workflows → Import from File*).
2. Open the **KB Config — EDIT ME** node and set:
   - `qdrantUrl` → your cluster endpoint (`https://YOUR-CLUSTER.cloud.qdrant.io:6333`)
   - `collection` → `woocommerce-policies` (or your custom name)
3. Create/attach credentials:
   - **Qdrant API** credential (cluster URL + API key) on the two HTTP nodes
     (*Ensure Doc-Title Index*, *Remove Previous Version*) and on
     **Qdrant — Knowledge Base**.
   - **OpenAI** credential on **Embeddings OpenAI1** — model
     **`text-embedding-3-small`**.
4. **Protect the form**: open the **Upload Form** node → *Credential for Basic Auth* →
   create a username + password. Without this, anyone with the URL could inject documents
   that steer the assistant's answers.
5. **Activate** the workflow.

## 3. Upload a policy document

1. Open the **Upload Form** node and copy its **Production URL**
   (looks like `https://your-n8n/form/222ad0c9-...`).
2. Open it in a browser, log in with the Basic Auth credentials you created.
3. Fill in:
   - **Document title** — e.g. `Return policy`. Re-using an existing title replaces the
     old version.
   - **Category** — Policies / About us / Shipping & delivery / Returns & refunds /
     Payment / FAQ / Other.
   - **Document** — the file (`.pdf`, `.docx`, `.txt`, `.md`, `.csv`).
4. Submit — the confirmation page shows how many chunks were stored.

The Sales Agent's `search_store_info` tool can now answer from this document.

## Tips

- Write documents in clear, self-contained paragraphs — the agent answers only from what
  it retrieves.
- French (or any language) documents are fine; the agent translates answers into the
  customer's language.
- To remove a document entirely, delete its points by title:

```bash
curl -X POST "https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-policies/points/delete?wait=true" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "filter": { "must": [ { "key": "metadata.doc_title", "match": { "value": "Return policy" } } ] } }'
```
