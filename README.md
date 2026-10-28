# Voice-Enabled E-commerce Assistant (MCP + RAG)

A demo e-commerce platform with a browser voice assistant, text/voice interaction, a live local product catalog, order creation and order lookup, local Retrieval-Augmented Generation (RAG), and an MCP server for compatible AI clients. Hindi, English, and Hinglish are supported by the configured speech/LLM services.

## Features
- Product search by name, SKU, category, or feature; live demo prices and inventory.
- Voice assistant and text assistant using the existing speech pipeline.
- Order creation with explicit customer confirmation before writing an order.
- Order reference generation (`ORD-XXXXXX`) and order-status lookup.
- SQLite-backed demo catalog and orders (`ecommerce.sqlite3` is created automatically).
- Local RAG over approved store policy documents in `knowledge_base/`.
- MCP tools for product search/details, order placement/tracking, and store knowledge retrieval.
- Existing latency benchmarks and tests retained where applicable.

## Quick start (Windows PowerShell)
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
# Edit .env and add the required Groq/ElevenLabs credentials for cloud speech/LLM features.
python server.py
```
Open `http://localhost:8000`.

The demo catalog includes headphones, a USB-C charger, a fitness watch, a cotton T-shirt, a backpack, a water bottle, and a desk lamp. Prices and stock are sample data. SQLite is initialized automatically.

## RAG
`knowledge_base.py` provides lightweight local lexical retrieval without a paid embedding API or external vector database. Store policy documents are in `knowledge_base/store_faq.md`. Add approved `.md` or `.txt` documents to that folder. Test retrieval at:
`http://localhost:8000/api/knowledge/search?q=shipping%20policy`

## REST endpoints
- `GET /api/products?q=headphones&category=Electronics` — search catalog.
- `GET /api/products/{sku}` — product details.
- `GET /api/orders/{order_ref}` — look up an order.
- `GET /api/orders` — list recent demo orders; protect this endpoint before public deployment.
- `GET /api/knowledge/search?q=returns` — search local store policy knowledge.
- `POST /api/orders` — create an order; request JSON includes `customer_name`, `address`, `items` and `confirmed: true`.
- `POST /api/turn` — existing voice/text conversation endpoint.

Example order body:
```json
{
  "customer_name": "Demo Customer",
  "address": "123 Example Road, Jaipur",
  "items": [{"sku": "SKU1001", "quantity": 1}],
  "confirmed": true
}
```

## MCP server
Run in a separate terminal:
```powershell
python mcp_server.py
```
The server uses stdio transport and exposes `search_products`, `get_product_details`, `place_order`, `track_order`, `search_store_knowledge`, and `list_recent_orders`. Configure it in your MCP-compatible client using the correct Python interpreter and absolute path to `mcp_server.py`.

## Important demo limitations
- Product prices, inventory, and order records are local sample data.
- No payment gateway, courier, live shipping tracking, email/SMS notifications, or real automated returns/refunds are connected. The indexed return/refund policy is illustrative demo content only.
- Do not collect card numbers, CVV, passwords, or OTPs.
- The recent-orders endpoint and MCP listing tool do not implement user authentication. Do not expose them publicly without adding authentication and authorization.
- For production, add customer authentication, role-based access control, payment-provider integration, tax/shipping calculations, inventory concurrency controls, audit logging, database migrations, rate limiting, HTTPS, and privacy/legal review.

## Tests
```powershell
python -m pytest tests -q
```
Some legacy appointment-specific tests may no longer apply because the domain workflow has been replaced with e-commerce operations. New e-commerce tests are in `tests/test_ecommerce.py`.

## Local RAG documents and token-saving route

Approved store documents can be placed in `knowledge_base/` as `.pdf`, `.md`, or `.txt` files. `ShopMate_Store_Guide.pdf` includes a sample return/refund policy for demo and retrieval testing only; it is not an actual merchant policy and must be replaced before production use. The pipeline extracts text (PDF extraction uses `pypdf`), splits long text into bounded overlapping chunks, builds a cached local lexical index, ranks passages, and returns source-labelled matches. It does not require a paid embedding API or external vector database.

- `GET /api/knowledge/status` shows indexed documents and chunk counts.
- `POST /api/knowledge/ingest` forces a re-index after document changes.
- `GET /api/knowledge/search?q=return%20policy` searches approved documents.

For high-confidence shipping, return, payment, and privacy questions, ShopMate can answer directly from the best retrieved passage without calling the LLM. If retrieval is weak, the normal tool/LLM workflow remains available. Clearly unrelated general-knowledge questions are redirected without an LLM request. Live prices, stock, and order status must still come from the live catalog/order tools, not the document index.

Latency records include the `rag_seconds` and `answer_route` fields. A `rag_direct` route means the answer was served from the approved local documents without an LLM call; `scope_redirect` means a fixed scope response was used.
