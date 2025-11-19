# Variables & Code Node Contracts (Main Workflow)

This document standardizes all variable names and describes required inputs/outputs
for each code node in the **Elanus 666 Amazon → Inventory → Google Sheets** pipeline.

---

## 0. Unified Variable Naming

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `region` | string | `"EU"` | Logical region for marketplace |
| `host` | string | `"https://sellingpartnerapi-eu.amazon.com"` | SP-API base URL |
| `awsRegion` | string | `"eu-west-1"` | AWS sig region |
| `marketplaceIds` | string[] | `["A1F83G8C2ARO7P"]` | SP-API filter |
| `createdAfter` | string | ISO timestamp | Time filter |
| `lwaAccessToken` | string | LWA token | Passed as header |
| `AmazonOrderId` | string | `204-7656641-3101144` | Order ID |
| `MarketplaceId` | string | Market ID | Defines region |
| `OrderStatus` | string | `"unshipped"` | Lowercase |
| `SellerSKU` | string | `"D188"` | Internal SKU |
| `delta` | object | `{ stock: -1, reserved: 1, sold: 0 }` | Inventory change |

---

## 1. Define_marketplace (Region Splitting)

### Input
```json
{
  "marketplaceIds": ["A1F83G8C2ARO7P","ATVPDKIKX0DER"],
  "createdAfter": "2025-11-06T17:15:02.981Z",
  "lwaAccessToken": "Atza|IwE..."
}
```

### Output
```json
[
  {
    "host": "https://sellingpartnerapi-eu.amazon.com",
    "awsRegion": "eu-west-1",
    "marketplaceIds": ["A1F83G8C2ARO7P"],
    "createdAfter": "2025-11-06T17:15:02.981Z",
    "lwaAccessToken": "Atza|IwE..."
  },
  {
    "host": "https://sellingpartnerapi-na.amazon.com",
    "awsRegion": "us-east-1",
    "marketplaceIds": ["ATVPDKIKX0DER"],
    "createdAfter": "2025-11-06T17:15:02.981Z",
    "lwaAccessToken": "Atza|IwE..."
  }
]
```

---

## 2. AddRegionHost (Derive Region + Host)

### Input
```json
{
  "AmazonOrderId": "204-7656641-3101144",
  "MarketplaceId": "A1F83G8C2ARO7P",
  "OrderStatus": "Unshipped"
}
```

### Output
```json
{
  "AmazonOrderId": "204-7656641-3101144",
  "MarketplaceId": "A1F83G8C2ARO7P",
  "OrderStatus": "unshipped",
  "region": "EU",
  "host": "https://sellingpartnerapi-eu.amazon.com"
}
```

---

## 3. Items->Lines (Flatten OrderItems)

### Input
```json
{
  "AmazonOrderId": "204-7656641-3101144",
  "OrderStatus": "unshipped",
  "region": "EU",
  "Stock": 25,
  "payload": {
    "OrderItems": [
      {
        "SellerSKU": "D188",
        "ASIN": "B0861673BK",
        "Title": "Hanger",
        "QuantityOrdered": 1,
        "QuantityShipped": 0,
        "ItemPrice": { "Amount": 34.07, "CurrencyCode": "GBP" },
        "ItemTax": { "Amount": 6.82 }
      }
    ]
  }
}
```

### Output
```json
{
  "AmazonOrderId": "204-7656641-3101144",
  "MarketplaceId": "A1F83G8C2ARO7P",
  "OrderStatus": "unshipped",
  "region": "EU",
  "SellerSKU": "D188",
  "ASIN": "B0861673BK",
  "Title": "Hanger",
  "Stock": 25,
  "QuantityOrdered": 1,
  "QuantityShipped": 0,
  "ItemPriceAmount": 34.07,
  "ItemTaxAmount": 6.82,
  "ShippingPriceAmount": 0,
  "CurrencyCode": "GBP"
}
```

---

## 4. Decide (DecideActions – Per SKU Logic)

### Input
```json
{
  "AmazonOrderId": "204-7656641-3101144",
  "SellerSKU": "D188",
  "region": "EU",
  "OrderStatus": "unshipped",
  "QuantityOrdered": 1,
  "QuantityShipped": 0
}
```

### Output Types

#### 4.1 product_adjust
```json
{
  "op": "product_adjust",
  "SellerSKU": "D188",
  "region": "EU",
  "amazonOrderId": "204-7656641-3101144",
  "delta": {
    "stock": -1,
    "reserved": 1,
    "sold": 0
  }
}
```

#### 4.2 sale_log
```json
{
  "op": "sale_log",
  "AmazonOrderId": "204-7656641-3101144",
  "SellerSKU": "D188",
  "Qty": 1,
  "region": "EU",
  "CurrencyCode": "GBP",
  "Price": 34.07,
  "Tax": 6.82
}
```

#### 4.3 amazon_sync
```json
{
  "op": "amazon_sync",
  "SellerSKU": "D188",
  "fromRegion": "EU",
  "targetRegion": "NA",
  "stock": -1
}
```

---

## 5. Variables Required by all Branches

| Variable | Source | Used In | Description |
|---------|---------|---------|-------------|
| `lwaAccessToken` | LWA token refresh HTTP call | All SP-API HTTP calls | Access token |
| `marketplaceIds` | SET_variables | Region split | Defines fetch targets |
| `host` | AddRegionHost | OrderItems, StockUpdate | SP-API base URL |
| `region` | AddRegionHost | Decide, Sync | EU / NA / FE |
| `awsRegion` | Define_marketplace | AWS signing | Required for SP-API |
| `Stock` | GoogleSheets read | Items->Lines | Current inventory |
| `delta` | Decide | Stock updater | Inventory change |

---

## 6. Recommended Final Naming Standard

All future nodes should use:

### Order-level
```
AmazonOrderId
MarketplaceId
PurchaseDate
OrderStatus  // lowercase
region
host
awsRegion
```

### Item-level
```
SellerSKU
ASIN
Title
QuantityOrdered
QuantityShipped
ItemPriceAmount
ItemTaxAmount
ShippingPriceAmount
CurrencyCode
```

### Inventory
```
Stock
delta.stock
delta.reserved
delta.sold
```

---

## 7. Example End-to-End Flow (Abbreviated)

1. **LWA Token Refresh** → outputs `lwaAccessToken`  
2. **SET_variables** → prepares marketplace list  
3. **Define_marketplace** → splits into EU/NA/FE  
4. **ListOrders** (per region)  
5. **AddRegionHost**  
6. **orderItems fetch**  
7. **Items->Lines** (normalize per SKU)  
8. **Decide** (inventory delta, sale logs, cross-region sync)  
9. **Switch** →  
   - → Stock Update  
   - → Google Sheets Append  

---

This file now serves as the **canonical data contract** for the entire workflow.

