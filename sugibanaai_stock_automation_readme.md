# SugibanaAI — Amazon Stock Management Automation (n8n)

---

## 🧭 Overview

**SugibanaAI Stock Management Automation** is an end-to-end Amazon SP‑API integration built on **n8n** to manage, synchronize, and analyze product stock levels across multiple Amazon regions (EU, NA, FE). The workflow automates the collection of order data, updates product stock and reserved quantities in real-time, and prepares the foundation for alerting, forecasting, and replenishment automation.

Developed in collaboration with **Elanus** as an initial pilot customer, this system will evolve into a scalable SaaS solution under the **SugibanaAI** brand — targeting small to mid‑size eCommerce operators who need affordable, customizable, and transparent inventory automation.

---

## ⚙️ Architecture Summary

### 🧩 Core Workflow Logic

```mermaid
graph TD
    A[Amazon SP‑API Token Refresh] --> B[Define Marketplace Groups (EU / NA / FE)]
    B --> C[List Orders by Region]
    C --> D[Merge EU/NA/FE Orders]
    D --> E[Prepare Orders for Loop]
    E --> F[Filter Order Status: Unshipped / Shipped / Canceled]
    F --> G[DecideActions (JS logic)]
    G --> H[Switch by op: product_adjust / sale_log / amazon_sync / needs_items]
    H --> I[Update Google Sheets / DB]
    I --> J[Alerting & Reports]
```

### 🔑 Key Components

| Node | Description |
|------|--------------|
| **HTTP Request (Token)** | Authenticates via Amazon SP‑API using OAuth2 refresh token. |
| **Define Marketplace** | Groups marketplace IDs into EU / NA / FE regions dynamically. |
| **List Orders (EU/NA/FE)** | Pulls recent orders by region, including status filters. |
| **DecideActions (Code Node)** | Core logic that interprets order statuses and calculates SKU adjustments, reservation, and synchronization deltas. |
| **Switch Node** | Routes output into four logical categories (`product_adjust`, `sale_log`, `amazon_sync`, `needs_items`). |
| **Google Sheets / DB Nodes** | Store and reflect live stock, reserved quantities, and sales history. |
| **Alerting Modules** | Triggered when low stock or sync discrepancies occur. |

---

## 🧮 1. Main Stock Management Functionality

### 🔁 Order Intake & Normalization
- Fetches all **Amazon Orders** via the SP‑API `GET /orders/v0/orders` endpoint.
- Consolidates data from all connected regions.
- Normalizes key attributes: `AmazonOrderId`, `MarketplaceId`, `OrderStatus`, `PurchaseDate`, `OrderItems`.

### 🧠 Intelligent Status Handling
Each order is processed through the **DecideActions** node:

| Order Status | System Action | Inventory Impact |
|---------------|----------------|------------------|
| `Unshipped` | Reserve stock | Decrease available stock, increase reserved count |
| `Shipped` | Finalize sale | Decrease reserved, log sale to ledger |
| `Canceled` | Restore stock | Add stock back, decrease reserved |

Additionally:
- Prevents double-reservation via **status transition logic** (`LastKnownStatus` check).
- Supports **multi-region synchronization**, ensuring changes in one region propagate to others.

### 🧾 Outputs
Each order emits one or more structured operations:
- `product_adjust`: stock delta updates
- `sale_log`: structured sale entries
- `amazon_sync`: inter‑region synchronization
- `needs_items`: signals missing item detail retrieval

---

## 🔔 2. Alerting, Monitoring & Extensions

### 📉 Low‑Stock Alerts
Triggered when available stock ≤ threshold (e.g., 5 units).
Delivered via:
- Email summary (HTML table style)
- Telegram notification (real-time)

### 🧾 Daily Reports
Aggregated Google Sheet log (e.g., `Sales_Ledger`, `Product_Details`):
- Columns: SKU | Total Sold | Current Stock | Last Order | Region | Status
- Automatic daily export / backup at **22:30 Vilnius time**

### 💌 Order Updates
- Detects status transitions (e.g., `Unshipped → Shipped`) and sends confirmation email to business operators.
- Planned: Amazon/eBay order merge view dashboard (with JWT-based client login).

---

## 🚀 3. Roadmap & Competitive Edge vs Helium10

### 🧭 Near-Term (Q1–Q2 2026)
| Feature | Description | Business Impact |
|----------|--------------|-----------------|
| **Stock Replenishment Alerts** | Automated prediction of restock timing using moving average of daily sales velocity. | Proactive inventory control |
| **Auto‑Ordering System** | Direct integration with supplier API or Google Sheet to create purchase requests automatically. | Saves time & avoids out‑of‑stock losses |
| **Forecasting Module** | AI‑driven demand prediction per SKU. | Strategic reordering |
| **Region Expansion (NA, FE)** | Full multi‑region support via SP‑API endpoints for North America & Far East. | Global reach |
| **eBay & Custom API Support** | Synchronize stock across non‑Amazon channels (e.g., eBay, Shopify, custom API). | Multi‑channel unification |

### 💡 Long‑Term Differentiators (vs Helium10)
| SugibanaAI | Helium10 |
|-------------|-----------|
| Customizable automation with n8n + open API | Closed SaaS with limited automation |
| Self‑hostable & white‑label options | Cloud‑only model |
| Multi‑region stock sync (EU, NA, FE) | Market insights only |
| Integration with Google Sheets | No direct sheet integration |
| Modular alerting via Email/Telegram | Basic alerts only |
| AI‑based restock forecasting (planned) | N/A |
| Flexible pricing per SKU or order volume | Flat subscription tiers |

---

## 🧰 4. Setup & Deployment

### 🔑 Prerequisites
- **n8n v1.105+** (self‑hosted or cloud)
- **Amazon SP‑API credentials** (client ID, client secret, refresh token)
- (Optional) **Google Service Account** for Sheets access
- (Optional) **SMTP / Telegram Bot API key** for notifications

### ⚙️ Setup Steps
1. Import workflow JSON → `SugibanaAI_Stock_Automation.json`.
2. Open **Credentials** → Create entries for:
   - Amazon SP‑API (AWS Signature + LWA token)
   - Google Sheets (if applicable)
3. Edit **SET_variables** node → insert correct `marketplaceIds` and regions.
4. Add CRON trigger → e.g. every 30 min.
5. Test manually using `Execute Workflow`.

### 🔒 Security & Compliance
- OAuth tokens are stored securely in **n8n Credentials Vault**.
- No local plaintext secrets.
- All API communications use HTTPS with signed headers.
- Fully GDPR compliant (no PII stored beyond order meta data).

---

## 🧩 5. Maintenance & Troubleshooting

| Action | Description |
|---------|--------------|
| **Refresh tokens** | Re-run LWA token node if authentication expires. |
| **Check workflow logs** | n8n Execution log → search for failed HTTP nodes. |
| **API rate limits** | Respect 5 req/sec per region (use delays if needed). |
| **Error recovery** | The `needs_items` operation captures incomplete orders for reprocessing. |

---

## 🌍 6. Future Vision

SugibanaAI’s next evolution is a full **AI‑assisted inventory orchestration system**, providing:
- Multi‑platform order ingestion (Amazon, eBay, custom APIs)
- Predictive analytics for replenishment
- Cost tracking per SKU and marketplace
- Integrated web dashboard with multi‑client JWT authentication

---

### 📈 Conclusion

The **SugibanaAI n8n Stock Management System** offers a transparent, modular, and automation‑first alternative to closed tools like Helium10. Its open architecture enables continuous extension — empowering small eCommerce operators to achieve enterprise‑grade stock automation with accessible, cost‑efficient technology.

