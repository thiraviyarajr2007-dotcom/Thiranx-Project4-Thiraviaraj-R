# 🛒 End-to-End Retail Analytics Project

Discover **sales trends, top products, high-value customers and product pairings** from retail transaction data using **Python, Pandas, NumPy and Matplotlib**.

This project solves a typical business problem faced by retail and e-commerce companies: *"What is selling, who is buying, and when?"* The answers help optimize inventory, target valuable customers and increase sales.

> 📖 **Written to teach.** Every concept, formula, result and pitfall is explained below, so you can learn the whole project from this README without opening the notebook.

---

## 📑 Table of Contents

1. [Business problem](#-business-problem)
2. [Datasets used](#-datasets-used)
3. [Project roadmap](#-project-roadmap)
4. [Part 1: Real transaction data](#part-1--real-transaction-data-uci-online-retail-ii)
5. [Part 2: Product catalog and orders (joins)](#part-2--product-catalog--orders-synthetic-joining-tables)
6. [Part 3: Customer value and segmentation](#part-3--customer-value--segmentation)
7. [Part 4: Time trends](#part-4--time-trends)
8. [Part 5: Market basket analysis](#part-5--market-basket-analysis)
9. [Part 6: Data quality and common mistakes](#part-6--data-quality--common-mistakes)
10. [Mini project: top-selling category](#-mini-project-find-the-top-selling-category)
11. [Key results at a glance](#-key-results-at-a-glance)
12. [Cheat sheet](#-retail-analytics-cheat-sheet)
13. [Run it yourself](#-run-it-yourself)
14. [Credits](#-credits)

---

## 🎯 Business problem

A retailer has thousands of transactions but no clear picture. This project answers:

| Business question | Technique used |
|---|---|
| How much revenue did we make? | `Quantity × Price`, then sum |
| How many customers do we have? | Count unique IDs |
| What are our best-selling products? | Group, sum, sort |
| Which category earns the most? | Join tables, then group |
| Who are our top customers? | Group by customer, rank |
| How do sales change over time? | Daily/monthly totals, moving average |
| Which products are bought together? | Pair counting (market basket) |
| Is our data trustworthy? | Missing values and common-mistake checks |

---

## 📦 Datasets used

The project uses **one real dataset and three synthetic ones** (generated with a fixed random seed so results are reproducible).

| Name | Type | Size | Columns | Purpose |
|---|---|---|---|---|
| `df_transactions` | **Real**: UCI *Online Retail II* (sheet `Year 2010-2011`) | **541,910 × 8** | Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country | Real revenue, products, customers, daily trend |
| `df_products` | Synthetic | 100 × 3 | ProductID (1001–1100), Category, Price ($5–$500) | Product catalog |
| `df_orders` | Synthetic | 1,000 × 5 | OrderID, CustomerID, ProductID, Quantity (1–4), OrderDate (hourly from 2023-01-01) | Orders to join with the catalog |
| `df_basket` | Synthetic | 900 × 2 | TransactionID, Product | Market basket demo (300 baskets × 3 items, 8 grocery products) |

> 💡 **Why mix real and synthetic data?** Real data shows realistic messiness. Synthetic data lets you practise multi-table concepts (joins, categories) that the real file doesn't have.

---

## 🗺️ Project roadmap

```
 Load data ──► Revenue & customers ──► Top products ──► Join tables
                                                            │
   Business recommendation ◄── Basket analysis ◄── Time trends ◄── Customer segments
```

---

## Part 1 · Real Transaction Data (UCI Online Retail II)

### Core concept: the "grain" of a dataset

Retail data usually has **one row per item purchased on an invoice**. So one invoice can span several rows. Always ask: *"What does one row represent?"* before aggregating, because the grain decides how you group.

### 1.1 Load and parse dates

```python
df_transactions = pd.read_excel(url, sheet_name='Year 2010-2011')
df_transactions['InvoiceDate'] = pd.to_datetime(df_transactions['InvoiceDate'])
```

`pd.to_datetime` ensures dates are real datetime objects, which unlocks `.dt.date`, `.dt.month`, resampling and rolling windows.

### 1.2 Total revenue

There's no revenue column, so you **create** one:

```python
df_transactions['TotalSales'] = df_transactions['Quantity'] * df_transactions['Price']
df_transactions['TotalSales'].sum()      # → 9,747,765.93
```

> **Formula:** `Revenue per line = Quantity × Unit Price`

### 1.3 Unique customers

```python
df_transactions['Customer ID'].nunique()   # → 4,372
```

`nunique()` counts distinct values and **ignores missing IDs** (see data quality below).

### 1.4 Daily sales

```python
daily_sales = df_transactions.groupby(df_transactions['InvoiceDate'].dt.date)['TotalSales'].sum()
```

First week: Dec 1 → 58,635.56 · Dec 2 → 46,207.28 · Dec 3 → 45,620.46 …

### 1.5 Top 5 products by revenue

```python
df_transactions.groupby('Description')['TotalSales'].sum() \
               .sort_values(ascending=False).head(5)
```

| Rank | Product | Revenue |
|---|---|---|
| 1 | DOTCOM POSTAGE | 206,245.48 |
| 2 | REGENCY CAKESTAND 3 TIER | 164,762.19 |
| 3 | WHITE HANGING HEART T-LIGHT HOLDER | 99,668.47 |
| 4 | PARTY BUNTING | 98,302.98 |
| 5 | JUMBO BAG RED RETROSPOT | 92,356.03 |

> ⚠️ **Analyst's eye:** "DOTCOM POSTAGE" reads like a **postage/shipping charge, not merchandise**. A real analysis would exclude such non-product lines before ranking products. The notebook doesn't filter them, so they appear in the ranking.

---

## Part 2 · Product Catalog + Orders (Synthetic): Joining Tables

Real businesses store data in **separate tables** (orders in one, product details in another). You combine them with a **join (merge)**.

```python
df_merged = pd.merge(df_orders, df_products, on='ProductID', how='left')
df_merged['OrderValue'] = df_merged['Quantity'] * df_merged['Price']
```

| Concept | Explanation |
|---|---|
| **Key** | The shared column (`ProductID`) that links the two tables |
| **`how='left'`** | Keep **every order**, and attach product info where a match exists |
| **`OrderValue`** | Quantity × Price: the revenue of each order |

### Revenue by category

```python
df_merged.groupby('Category')['OrderValue'].sum().sort_values(ascending=False)
```

| Category | Revenue | Share |
|---|---|---|
| **Sports** | 147,783.07 | **24.89%** |
| Beauty | 134,105.61 | 22.59% |
| Clothing | 121,487.00 | 20.46% |
| Home | 98,098.72 | 16.52% |
| Electronics | 92,256.84 | 15.54% |

**Revenue share** = `category revenue ÷ total revenue`, computed as `category_sales / category_sales.sum()`.

---

## Part 3 · Customer Value & Segmentation

### 3.1 Spend per customer

```python
customer_order_value = df_merged.groupby('CustomerID')['OrderValue'].sum()
customer_order_value.mean()      # → 1,403.62
```

Average **total spend per customer**: 1,403.62.

> 📝 The notebook names this variable `avg_order_value`, but it's really *average spend per customer*. **Average Order Value (AOV)** would be total revenue ÷ number of *orders*. Mind the naming.

### 3.2 Top customers (the "VIPs")

```python
customer_order_value.sort_values(ascending=False).head(3)
```

| Customer ID | Total spend |
|---|---|
| 1098 | 5,808.86 |
| 1053 | 5,621.52 |
| 1222 | 4,999.08 |

Why it matters: a small group of customers often drives a large share of revenue (loyalty offers, early access, personalized outreach).

### 3.3 Spend segments with quartiles

```python
segments = pd.qcut(customer_order_value, 4, labels=['Low', 'Mid-Low', 'Mid-High', 'High'])
```

`qcut` splits customers into **4 equal-sized groups by spend rank** (quartiles):

| Segment | Customers |
|---|---|
| Low | 106 |
| Mid-Low | 106 |
| Mid-High | 105 |
| High | 106 |

Counts are nearly equal **by design**, since quartiles split by *rank*. (`pd.cut` would split by equal *value ranges* instead.) This is a simple, transparent first step toward customer segmentation.

---

## Part 4 · Time Trends

### 4.1 Extract time parts

```python
df_merged['Month'] = df_merged['OrderDate'].dt.month
monthly_sales = df_merged.groupby('Month')['OrderValue'].sum()
```

| Month | Revenue |
|---|---|
| 1 | 445,700.41 |
| 2 | 148,030.83 |

A line plot with markers (`plt.plot(..., marker='o')`) visualizes the trend.

> ⚠️ **Don't misread this as a sales crash.** The synthetic orders are generated **hourly**, 1,000 of them, which covers only about **41.7 days**: all of January plus roughly the first 11 days of February. February simply has **fewer days of data**, so its total is lower. Always check *how much time each period covers* before comparing totals. (Compare **daily averages** instead.)

### 4.2 7-day moving average (smoothing)

```python
rolling_avg = daily_sales.sort_index().rolling(window=7).mean()
```

A **moving average** replaces each day's value with the average of the last 7 days, smoothing out day-to-day noise to reveal the trend.

| Date | 7-day avg |
|---|---|
| 2010-12-01 … 12-07 | NaN (not enough history yet) |
| 2010-12-08 | 46,422.33 |
| 2010-12-09 | 45,550.41 |
| 2010-12-10 | 47,150.07 |

- The first **6 values are `NaN`** because a 7-day window needs 7 data points.
- Always `sort_index()` first. Rolling windows assume chronological order.
- ⚠️ `window=7` means **7 rows**, not 7 calendar days. In this data some dates (e.g. Dec 4 and Dec 11) have no sales, so those rows don't exist and the window stretches over more than 7 calendar days. Use `rolling('7D')` on a datetime index for true calendar windows.

---

## Part 5 · Market Basket Analysis

**Question:** *Which products are bought together?* Retailers use this for shelf placement, bundles and "customers also bought" recommendations.

### 5.1 Most frequently bought items

```python
df_basket.groupby('Product').size().sort_values(ascending=False)
```

Eggs 126 · Cheese 117 · Milk 117 · Apples 116 · Chicken 115 · Rice 108 · Bread 104 · Butter 97

### 5.2 Count product pairs

```python
from itertools import combinations
basket_sets = df_basket.groupby('TransactionID')['Product'].apply(set)

pair_counts = {}
for basket in basket_sets:
    for pair in combinations(sorted(basket), 2):
        pair_counts[pair] = pair_counts.get(pair, 0) + 1
```

How it works:

1. `set` turns each basket into **unique items** (duplicates collapse)
2. `combinations(sorted(basket), 2)` generates every **2-item pair** in the basket
3. `sorted` makes `(Bread, Milk)` and `(Milk, Bread)` count as the **same pair**
4. A dictionary tallies how often each pair appears

**Top pairs:**

| Pair | Baskets |
|---|---|
| Chicken + Eggs | 32 |
| Eggs + Rice | 32 |
| Bread + Eggs | 32 |
| Milk + Rice | 28 |
| Apples + Milk | 28 |

> ⚠️ This basket data is **randomly generated**, so these "associations" are just noise. They demonstrate the *method*, not real buying habits. On real data you'd go further with **support, confidence and lift** (see *Next steps*).

---

## Part 6 · Data Quality & Common Mistakes

The notebook deliberately shows mistakes beginners make.

### ❌ Mistake 1: Ignoring missing data

```python
df_transactions['Customer ID'].isnull().sum()    # → 135,080
```

**135,080 of 541,910 transactions (~25%) have no Customer ID.** Any customer-level analysis (spend per customer, segments, repeat buyers) silently excludes a quarter of the data. Revenue totals still include those rows, but customer counts don't.

### ❌ Mistake 2: Summing price without multiplying by quantity

```python
df_transactions['Price'].sum()                   # → 2,498,821.97   ← WRONG
```

| Method | Result |
|---|---|
| ❌ `Price.sum()` | 2,498,821.97 |
| ✅ `(Quantity × Price).sum()` | **9,747,765.93** |

The wrong method understates revenue by about **74%**, because it treats a 12-unit order the same as a 1-unit order.

### ❌ Mistake 3: Grouping at the wrong level

```python
df_merged.groupby('ProductID').size()
# {1001: 13, 1002: 8, 1003: 11, ...}
```

Question: *"orders per category"*. This gives orders **per product**. Group by `Category` to answer the question actually asked.

### ✅ Beginner checklist

- Know what one row represents
- Create the revenue column (`Quantity × Price`) before summing
- Check missing values **before** analysis
- Group at the level the question asks for
- Compare periods of equal length
- Watch for non-product rows (postage, fees) and returns/cancellations. The notebook doesn't filter these

---

## 🏆 Mini Project: Find the Top-Selling Category

**Goal:** identify which category drives the most revenue and make a business recommendation.

```python
category_revenue = df_merged.groupby('Category')['OrderValue'].sum().sort_values(ascending=False)
top_category = category_revenue.index[0]
top_revenue  = category_revenue.iloc[0]
```

**Result:** 🥇 **Sports**, with revenue of **147,783.07** (24.89% of total).

**Sample recommendation:** focus marketing and stock planning on **Sports**, while investigating why **Electronics** (15.54%) lags, with options like promotions, pricing review or bundling with top categories.

> The category data is synthetic, so treat this as a demonstration of the *workflow*. With real data, also compare **margin** and **units sold**, not just revenue.

---

## 📊 Key Results at a Glance

| Metric | Value |
|---|---|
| Transactions analysed | 541,910 |
| Total revenue | **9,747,765.93** |
| Unique customers | **4,372** |
| Transactions missing Customer ID | 135,080 |
| Top real "product" by revenue | DOTCOM POSTAGE (likely not merchandise) |
| Top real merchandise | REGENCY CAKESTAND 3 TIER (164,762.19) |
| Top category (synthetic) | Sports, 24.89% |
| Avg spend per customer (synthetic) | 1,403.62 |
| Top customer (synthetic) | #1098, 5,808.86 |
| Most common basket pair (synthetic) | Chicken + Eggs (32) |

---

## 🧾 Retail Analytics Cheat Sheet

| Task | Code |
|---|---|
| Revenue column | `df['Total'] = df['Quantity'] * df['Price']` |
| Total revenue | `df['Total'].sum()` |
| Unique customers | `df['Customer ID'].nunique()` |
| Missing values | `df['col'].isnull().sum()` |
| Top N by revenue | `df.groupby('X')['Total'].sum().sort_values(ascending=False).head(N)` |
| Daily totals | `df.groupby(df['Date'].dt.date)['Total'].sum()` |
| Join tables | `pd.merge(a, b, on='key', how='left')` |
| Share of total | `series / series.sum()` |
| Customer quartiles | `pd.qcut(series, 4, labels=[...])` |
| Month from date | `df['Date'].dt.month` |
| 7-day moving average | `series.sort_index().rolling(7).mean()` |
| Item pairs | `itertools.combinations(sorted(basket), 2)` |
| Line chart | `plt.plot(x, y, marker='o')` |

---

## ▶️ Run it yourself

### Requirements

- Python 3.8+
- pandas · numpy · matplotlib · **openpyxl** (needed by `pd.read_excel` for `.xlsx`) · jupyter

```bash
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>
pip install pandas numpy matplotlib openpyxl jupyter
jupyter notebook
```

### Notes

- The notebook reads the Excel file **directly from the UCI repository URL**. The first load is large (~540k rows) and can be slow. If the link is unavailable, download *Online Retail II* manually and use `pd.read_excel('online_retail_II.xlsx', sheet_name='Year 2010-2011')`.
- Synthetic data uses `np.random.seed(42)`, so results are reproducible. Results depend on running the cells in order.

---

## 🚀 Next steps

- **Clean the real data:** drop missing Customer IDs, remove postage/fee lines, handle negative quantities (returns)
- **RFM analysis** (Recency, Frequency, Monetary) for richer customer segments
- **Basket metrics:** support, confidence and lift for genuine association rules
- **Resample** to weekly/monthly with `df.resample('M')` and compare average daily sales
- **Visualize** category share (bar chart) and top products
- **Forecast** demand using the daily series

---

## 🙌 Credits

- Original lesson and notebook: **Mathew K Analytics**, *End to End Retail Analytics Project* ([website](https://www.mathewkanalytics.com) · [YouTube](https://www.youtube.com/watch?v=MEJFJHp4tiA))
- Real dataset: **UCI Machine Learning Repository**, *Online Retail II*
- Practised and documented by **Thiraviaraj R**

---

⭐ If this README helped you, consider starring the repo!
