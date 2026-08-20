# WooCommerce → Qdrant Synchronizer

Keeps your WooCommerce product catalog mirrored into a Qdrant vector collection so the
WhatsApp Sales Agent can search products semantically.

**Workflow file:** `Woocommerce - Qdrant Synchronizer.json`

## What it does

- Listens to WooCommerce webhooks: `product.created`, `product.updated`, `product.deleted`.
- Builds a compact embedding text per product (name, SKU, categories, tags, attributes,
  short description) and stores it in Qdrant with metadata:
  `product_id`, `sku`, `price`, `stock_status`, `permalink`.
- Updated products: old vectors are deleted first, then re-inserted (no duplicates).
- Deleted/unpublished products: their vectors are removed.
- A manual **FULL RESYNC — Click Execute** trigger re-indexes the entire catalog on demand.

## Prerequisites / subscriptions

| Service | What you need | Cost |
|---|---|---|
| n8n | A running instance (self-hosted or cloud) with the LangChain nodes | — |
| [Qdrant Cloud](https://cloud.qdrant.io/) | Free tier cluster (1 GB RAM is plenty to start) | Free |
| [OpenAI](https://platform.openai.com/) | API key for `text-embedding-3-small` embeddings | Pay-per-use (very cheap) |
| WooCommerce | REST API key with **Read/Write** permission | Free |

## 1. Create the Qdrant cluster and collection

1. Sign up at [cloud.qdrant.io](https://cloud.qdrant.io/) and create a **free cluster**.
2. Copy the cluster **endpoint URL** (looks like
   `https://xxxx.eu-central-1-0.aws.cloud.qdrant.io:6333`) and create an **API key**
   under *Data Access Control → API Keys*.
3. Create the products collection. It **must** use vector size **1536** with **Cosine**
   distance, because that is what OpenAI `text-embedding-3-small` produces:

```bash
curl -X PUT "https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-products" \
  -H "api-key: YOUR_QDRANT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 1536,
      "distance": "Cosine"
    }
  }'
```

> The collection name used throughout these workflows is **`woocommerce-products`**. If you pick a
> different name, update it everywhere (this workflow's Qdrant nodes and the Sales Agent's
> `search_products` tool).

Reference: [Qdrant collections docs](https://qdrant.tech/documentation/concepts/collections/).

## 2. Get an OpenAI API key (embeddings)

1. Create a key at [platform.openai.com/api-keys](https://platform.openai.com/api-keys).
2. In n8n: **Credentials → Add credential → OpenAI**, paste the key.
3. Open the **Embeddings OpenAI2** node in this workflow, select your credential and make
   sure the model is **`text-embedding-3-small`** (1536 dimensions — must match the
   collection above).

Docs: [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings).

## 3. Get WooCommerce API keys

1. In WordPress admin: **WooCommerce → Settings → Advanced → REST API → Add key**.
2. Description: `n8n sync`, User: an admin account, Permissions: **Read/Write**
   (n8n needs write access to register the product webhooks).
3. Copy the **Consumer Key** and **Consumer Secret**.
4. In n8n: create a **WooCommerce API** credential with your store URL
   (e.g. `https://your-store.com`), the consumer key and secret.
5. Attach this credential to all four WooCommerce nodes in this workflow
   (**Product Created**, **Product Updated**, **Product Deleted**, **Get many products**).

Docs: [WooCommerce REST API docs](https://woocommerce.github.io/woocommerce-rest-api-docs/).

## 4. Set your Qdrant cluster URL

Two HTTP Request nodes ship with a placeholder cluster URL
(`https://your-cluster-id.cloud.qdrant.io:6333`). Open them and replace the host with
**your** cluster endpoint (keep the path):

- **Delete Old Vectors**
- **Remove Vectors Only**

```
https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-products/points/delete?wait=true
```

Both use the **Qdrant API** credential — create one in n8n
(*Credentials → Add credential → Qdrant*) with your cluster URL and API key.

## 5. Import and start

1. In n8n: **Workflows → Import from File** → select `Woocommerce - Qdrant Synchronizer.json`.
2. Attach all credentials (Qdrant, OpenAI, WooCommerce) to their nodes.
3. **Activate** the workflow (toggle top-right). Activation registers the three product
   webhooks in WooCommerce — you can verify them under
   *WooCommerce → Settings → Advanced → Webhooks*.
4. Run the initial backfill: open the workflow, click **Execute workflow** — the manual
   **FULL RESYNC** trigger will pull every product and index it into Qdrant.

From then on, every product change in WooCommerce is synced automatically within seconds.

## Verify it works

```bash
curl "https://YOUR-CLUSTER.cloud.qdrant.io:6333/collections/woocommerce-products" \
  -H "api-key: YOUR_QDRANT_API_KEY"
```

`points_count` should match your published product count after the full resync.
