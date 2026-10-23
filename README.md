# StoreOS — In-Store Digital POS System

## Architecture

```
React (port 3000) → FastAPI (port 8000) → n8n (port 5678)
```

- **React** — Cashier-facing POS terminal (4 screens)
- **FastAPI** — Pure proxy, forwards requests to n8n
- **n8n** — Handles DB (PostgreSQL), invoice generation, email sending

---

## Folder Structure

```
pos-system/
├── backend/
│   ├── main.py              ← FastAPI app (2 endpoints)
│   ├── requirements.txt
│   ├── .env                 ← n8n webhook URLs
│   └── start.bat            ← Windows startup script
│
└── frontend/
    ├── src/
    │   ├── App.jsx           ← Router / screen manager
    │   ├── App.css           ← All styles
    │   ├── main.jsx          ← Entry point
    │   ├── context/
    │   │   └── CartContext.jsx  ← Global cart state + discount logic
    │   ├── data/
    │   │   └── products.js   ← Hardcoded products, prices, discounts
    │   └── pages/
    │       ├── Catalog.jsx       ← Screen 1: Browse + search products
    │       ├── Cart.jsx          ← Screen 2: Review cart + discounts
    │       ├── Checkout.jsx      ← Screen 3: Customer info + payment
    │       └── ResponseScreen.jsx ← Screen 4: n8n response + refund
    ├── index.html
    ├── package.json
    ├── vite.config.js
    └── start.bat            ← Windows startup script
```

---

## Setup & Run

### Step 1 — Start n8n (must be running first)
Make sure n8n is running at `http://localhost:5678`
- Order webhook:  `http://localhost:5678/webhook/order`
- Refund webhook: `http://localhost:5678/webhook/refund-order`

---

### Step 2 — Start FastAPI Backend

```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

**Or double-click** `backend/start.bat` on Windows.

FastAPI will be available at: `http://localhost:8000`
API docs at: `http://localhost:8000/docs`

---

### Step 3 — Start React Frontend

```bash
cd frontend
npm install
npm run dev
```

**Or double-click** `frontend/start.bat` on Windows.

React will be available at: `http://localhost:3000`

---

## FastAPI Endpoints

### `POST /order`
Forwards order to n8n and returns n8n response.

**Request body:**
```json
{
  "customer": {
    "name": "Arjun Sharma",
    "email": "arjun.sharma@example.com",
    "phone": "+91 98765 43210",
    "address": {
      "line1": "42 Brigade Road",
      "city": "Bengaluru",
      "state": "Karnataka",
      "pincode": "560001"
    }
  },
  "items": [
    { "sku": "LAPTOP-001", "quantity": 1 },
    { "sku": "MOBILE-001", "quantity": 2 }
  ],
  "payment_method": "razorpay",
  "notes": "Test order",
  "order_summary": {
    "line_items": [...],
    "total_product_discount": 7499.95,
    "total_tax": 18000.00,
    "header_discount_name": null,
    "header_discount_pct": 0,
    "header_discount_amount": 0,
    "grand_total": 154498.05
  }
}
```

---

### `POST /refund`
Forwards refund to n8n and returns n8n response.

**Request body:**
```json
{
  "invoice_number": "INV-20260408-254632",
  "reason": "Customer changed mind"
}
```

---

### `GET /health`
Returns `{"status": "ok"}` — use to verify FastAPI is running.

---

## React Screens

| Screen | What it does |
|---|---|
| **Catalog** | Browse all 20 products, search by name/SKU, filter by collection, add to cart |
| **Cart** | Review items, adjust quantities, remove items, see all discounts + GST |
| **Checkout** | Fill customer details + address, choose payment method (Razorpay/Stripe/COD), place order |
| **Response** | Shows n8n's full webhook response, option to process refund |

---

## Discount Logic (in React frontend)

### Product-Level Discounts (active in seed data)
| SKU | Type | Value |
|---|---|---|
| LAPTOP-001 | percentage | 10% off |
| LAPTOP-002 | fixed | ₹5,000 off |
| MOBILE-001 | percentage | 5% off |
| MOBILE-002 | fixed | ₹2,000 off |
| CABLE-USB-C | fixed | ₹100 off |

### Header-Level Discounts (date-based, auto-applied)
| Name | Discount | Period |
|---|---|---|
| Independence Day Sale | 8% | Aug 14–15, 2026 |
| Diwali Sale | 15% | Oct 20–25, 2026 |
| New Year Sale | 10% | Dec 31–Jan 2, 2027 |

Header discounts are automatically applied when the current date falls within the range.

---

## Payload sent to n8n

The `order_summary` field contains pre-computed values so n8n doesn't need to recalculate:

- `line_items` — array with unit price, discount, tax rate, tax amount, line total per SKU
- `total_product_discount` — sum of all product-level discounts
- `total_tax` — total GST amount
- `header_discount_name/pct/amount` — header discount details (null if none active)
- `grand_total` — final payable amount

---

## Environment Variables (backend/.env)

```
N8N_ORDER_WEBHOOK=http://localhost:5678/webhook/order
N8N_REFUND_WEBHOOK=http://localhost:5678/webhook/refund-order
```

Change these if your n8n runs on a different host/port.
