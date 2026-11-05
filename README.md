# SENTINEL on MongoDB

Base Building Track — MongoDB with Python. The notebook loads SENTINEL fraud data from Supabase into
MongoDB Atlas, then demonstrates schema design, CRUD, indexing with `explain()`, and aggregation pipelines.

Everything is in one notebook: [`SENTINEL_MongoDB_Project.ipynb`](SENTINEL_MongoDB_Project.ipynb).

| # | Task | Notebook section |
|---|------|------------------|
| 1 | Design `customers` (devices embedded) and `transactions` (customer referenced) | 1. Schema design |
| 2 | Load ≥ 5,000 transactions with customer, merchant and device fields | 2. Load data from Supabase |
| 3 | `insert_one`, `insert_many`, `find` with filter + projection, `update_many`, `delete_one` | 3. CRUD operations |
| 4 | Indexes on `customer_id` and `transaction_time`, `explain()` before and after | 4. Indexes and explain() |
| 5 | Fraud rate per merchant category; top 10 customers by amount in their last 24 h | 5. Aggregation pipelines |

---

## Setup

**Prerequisites**
- Python 3.9+
- A MongoDB Atlas cluster (free M0 is enough) with your IP allowed under *Network Access*
- A Supabase project loaded with the SENTINEL data (Project Work 4/5: `sql/01_create_tables.sql` + `python ingest.py all`)

**1. Clone and install**
```bash
git clone https://github.com/<your-username>/sentinel-mongodb.git
cd sentinel-mongodb
python -m venv .venv
.venv\Scripts\activate          # Windows  (macOS/Linux: source .venv/bin/activate)
pip install -r requirements.txt
```

**2. Add your credentials**

Copy the template and fill in your own values:
```bash
copy .env.example .env          # Windows  (macOS/Linux: cp .env.example .env)
```

| Variable | Where to find it |
|---|---|
| `SUPABASE_URL` | Supabase dashboard → Project Settings → Data API |
| `SUPABASE_KEY` | Supabase dashboard → Project Settings → API Keys (publishable key — read access is enough) |
| `MONGODB_URI` | Atlas → Database → Connect → Drivers |
| `MONGODB_DATABASE` | Any name, e.g. `SENTINEL_DB` |

> `.env` is listed in `.gitignore` and is never committed. Only `.env.example` (placeholders, no real values) is in the repo.

## How to run

```bash
jupyter notebook SENTINEL_MongoDB_Project.ipynb
```

Then **Run All**. The notebook:
1. Reads `.env` and connects to Supabase and MongoDB Atlas.
2. Fetches 5,000 transactions plus their related customers, merchants, devices and labels from Supabase.
3. Builds the documents and writes them with `ReplaceOne(upsert=True)` — safe to re-run, no duplicates.
4. Runs the CRUD demo, the index/`explain()` comparison and the two aggregation pipelines.

---

## Schema design: embed or reference?

### `customers` — devices are **embedded**
```
customers { _id: customer_id, customer_age, home_city, region, created_at, is_synthetic,
            devices: [ { device_id, device_kind, is_synthetic, first_seen, last_seen, transaction_count } ] }
```
- **Small and bounded:** a customer uses a few phones or browsers, so the array stays far below the 16 MB document limit.
- **Read together:** "which devices does this customer use?" is answered by one read, no `$lookup`.
- **Owned by the customer:** devices have no life of their own beyond their kind; they are not updated separately.

The SENTINEL `devices` table has no `customer_id`, so the link comes from the transactions: each embedded device
records when the customer first and last used it and how many payments they made with it.

### `transactions` — customer is **referenced**; merchant and device are **embedded**
```
transactions { _id: transaction_id, customer_id (-> customers._id), merchant_id, device_id,
               merchant: { merchant_id, synthetic_name, merchant_category, merchant_location, is_synthetic },
               device:   { device_id, device_kind, is_synthetic },
               amount_paise, transaction_amount (Decimal128), payment_channel, transaction_time, ...,
               fraud_label }
```
- **Reference the customer** — transactions grow without limit; embedding them in the customer would hit the
  16 MB limit and make every customer read heavy. Fraud queries scan transactions by time, category and label
  across all customers, so they need to be top-level documents that can be indexed.
- **Embed merchant and device** — the fraud-rate pipeline groups by `merchant.merchant_category` without a join,
  and the copy records what was true at the moment of payment (what an audit needs).
- **Money as `Decimal128`** — exact rupee amounts, no float rounding.

---

## Output screenshots

### Connection
![Connected to Supabase and MongoDB Atlas](screenshots/01_connection.png)

### Data loaded from Supabase into MongoDB
![5,000 transactions loaded](screenshots/02_load.png)

### CRUD operations
![insert, find, update, delete](screenshots/03_crud.png)

### explain() before indexes — COLLSCAN
![COLLSCAN before indexes](screenshots/04_explain_before.png)

### explain() after indexes — IXSCAN
![IXSCAN after indexes](screenshots/04_explain_after.png)

### Fraud rate per merchant category
![Fraud rate per merchant category](screenshots/05_fraud_rate.png)

### Top 10 customers by amount in their last 24 hours
![Top 10 customers last 24h](screenshots/05_top_customers.png)

---

## Project structure
```
sentinel-mongodb/
├── SENTINEL_MongoDB_Project.ipynb   # all five sections
├── README.md
├── requirements.txt
├── .env.example                     # template — copy to .env
├── .gitignore                       # keeps .env out of git
└── screenshots/                     # output images used above
```

## Security
- No keys or passwords are in the code or the notebook; everything comes from `.env` via `python-dotenv`.
- Use the Supabase **publishable** key here, never the secret (service_role) key.
- If a credential is ever committed by mistake, rotate it immediately (Atlas → Database Access, Supabase → API Keys);
  removing it from git history is not enough.
