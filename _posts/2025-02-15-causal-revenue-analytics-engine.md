---
layout: post
title: Causal Revenue Analytics Engine - Period-over-Period Analysis
image: "/posts/causal-revenue-analytics-title.png"
tags: [LLM, Python, Llama, Data Analytics, Business Intelligence]
---

In this project I built an automated causal revenue analytics system that compares two time periods to identify why revenue changed, which products drove the change, and what actions to take. The system combines pandas-based decomposition with LLM-powered insights.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Business Problem](#overview-problem)
    - [Solution Architecture](#overview-solution)
    - [Key Features](#overview-features)
- [01. Two-Stage Architecture](#architecture)
- [02. Stage 1: Revenue Decomposition Engine](#stage1)
    - [Revenue Bridge Analysis](#bridge)
    - [Price-Volume-Mix Decomposition](#pvm)
    - [Quadrant Classification](#quadrants)
- [03. Stage 2: LLM Reporter](#stage2)
    - [Prompt Engineering](#prompts)
    - [Robust JSON Parsing](#parsing)
- [04. PVM Methodology Deep Dive](#pvm-deep)
- [05. Quadrant Logic & Actions](#quadrant-logic)
- [06. Example Output](#results)
- [07. Configuring English Output](#english-config)
- [08. Next Steps](#growth-next-steps)

___

# Project Overview  <a name="overview-main"></a>

<br>
### Business Problem <a name="overview-problem"></a>

Finance and business intelligence teams need to answer: **"Why did revenue change between two periods?"**

Traditional approaches either:
- Provide high-level totals without causality (dashboards showing +13M but not why)
- Require manual SQL queries and spreadsheet analysis (time-consuming, error-prone)
- Lack actionable recommendations (numbers without next steps)

The goal was to build an **automated system** that:
1. Decomposes revenue changes into **Price**, **Volume**, and **Mix** effects
2. Identifies **which products** drove each effect
3. Classifies products into **decision quadrants** (growth vs. decline)
4. Generates **executive summaries** in natural language
5. Provides **product-level actions** (keep, scale promo, delist, etc.)

<br>
### Solution Architecture <a name="overview-solution"></a>

The solution uses a **two-stage pipeline**:

**Stage 1: Pandas Analyzer** (`analyzer_from_csv_v2.py`)
- Loads transaction data from CSV
- Computes revenue bridge (appearing/disappearing/common products)
- Performs PVM decomposition on common products
- Classifies products into quadrants
- Computes margin impacts and basket metrics
- Outputs structured JSON

**Stage 2: LLM Reporter** (`llm_v2.py`)
- Reads JSON from Stage 1
- Uses Llama 3.1 (8B, quantized) to generate insights
- Renders human-readable report with tables

<br>
### Key Features <a name="overview-features"></a>

* **Revenue Bridge**: Separates appearing, disappearing, and common products
* **PVM Decomposition**: Price, Volume, and Mix effects
* **Quadrant Classification**: D1-D4 based on price/volume dynamics
* **Customer Segmentation**: Repeat vs. one-time buyer analysis
* **Margin Analysis**: Net margin tracking at product level
* **LLM Insights**: Natural language summaries and recommendations
* **Action Framework**: 5 decision buckets (KEEP, PROMO, PRICE TEST, FIX, DELIST)

___

# Two-Stage Architecture  <a name="architecture"></a>

```
Stage 1: Analyzer (Pure Computation)
    CSV Data
        ↓
    Revenue Bridge
    PVM Decomposition
    Quadrant Classification
        ↓
    llm_input_v2.json
        ↓
Stage 2: Reporter (LLM + Rendering)
    Load JSON
        ↓
    Generate Insights (LLM)
        ↓
    Render Report
        ↓
    weekly_summary_v2.txt
```

**Why two stages?**

1. **Auditability**: All math is deterministic (Stage 1)
2. **Modularity**: Can run Stage 1 alone for APIs
3. **LLM Independence**: Can swap or skip LLM
4. **Performance**: Stage 1 (~2-5s), Stage 2 (~10-15s)

___

# Stage 1: Revenue Decomposition  <a name="stage1"></a>

<br>
### Revenue Bridge Analysis  <a name="bridge"></a>

The bridge separates three product universes:

```python
def compute_presence(ct2_pack_weekly: pd.DataFrame) -> pd.DataFrame:
    p = ct2_pack_weekly.copy()
    p["in_t1"] = (p["q1"] != 0) | (p["r1"] != 0)
    p["in_t2"] = (p["q2"] != 0) | (p["r2"] != 0)
    return p[["CT2_pack", "in_t1", "in_t2"]]

def compute_revenue_bridge_summary(ct2_pack_weekly, presence):
    w = ct2_pack_weekly.merge(presence, on="CT2_pack")

    # Appearing: only in T2
    appearing = w.loc[(~w["in_t1"]) & (w["in_t2"]), "r2"].sum()

    # Disappearing: only in T1
    disappearing = w.loc[(w["in_t1"]) & (~w["in_t2"]), "r1"].sum()

    # Common: in both periods
    common_mask = (w["in_t1"]) & (w["in_t2"])
    common_delta = (w.loc[common_mask, "r2"] - w.loc[common_mask, "r1"]).sum()

    total_delta = appearing - disappearing + common_delta

    return {"appearing": appearing,
            "disappearing": disappearing,
            "common_delta": common_delta,
            "total_delta": total_delta}
```

**Bridge Formula:**
```
ΔR_total = R_appearing - R_disappearing + ΔR_common
```

<br>
### Price-Volume-Mix Decomposition  <a name="pvm"></a>

For common products, decompose revenue change:

```python
def compute_common_pvm(ct2_pack_weekly, presence):
    common = ct2_pack_weekly.merge(presence)
    common = common[(common["in_t1"]) & (common["in_t2"])]

    # Priced products (can compute unit prices)
    priced = common[(common["q1"] != 0) & (common["q2"] != 0) &
                    common["p1"].notna() & common["p2"].notna()]

    # Volume effect: p1 * (q2 - q1)
    volume = (priced["p1"] * (priced["q2"] - priced["q1"])).sum()

    # Price effect: q2 * (p2 - p1)
    price = (priced["q2"] * (priced["p2"] - priced["p1"])).sum()

    # Mix: residual
    delta_common = (common["r2"] - common["r1"]).sum()
    mix = delta_common - volume - price

    return {"volume": volume, "price": price, "mix": mix}
```

**PVM Formula:**
```
ΔR_common = Volume + Price + Mix

Volume = Σ[p1 * (q2 - q1)]  (quantity change at T1 price)
Price  = Σ[q2 * (p2 - p1)]  (price change at T2 quantity)
Mix    = residual           (product mix + unpriced)
```

<br>
### Quadrant Classification  <a name="quadrants"></a>

Classify products by price/volume direction:

```python
def compute_pack_quadrant(ct2_pack_weekly):
    w = ct2_pack_weekly.copy()

    def quad(row):
        p1, p2, q1, q2 = row["p1"], row["p2"], row["q1"], row["q2"]

        if pd.isna(p1) or pd.isna(p2):
            return "Neutral"

        if (p2 > p1) and (q2 > q1): return "D1: Price↑ Volume↑"
        if (p2 > p1) and (q2 < q1): return "D2: Price↑ Volume↓"
        if (p2 < p1) and (q2 > q1): return "D3: Price↓ Volume↑"
        if (p2 < p1) and (q2 < q1): return "D4: Price↓ Volume↓"
        return "Neutral"

    w["quadrant"] = w.apply(quad, axis=1)
    return w
```

**Quadrants:**

| Code | Price | Volume | Meaning | Action |
|------|-------|--------|---------|--------|
| D1 | ↑ | ↑ | Growth | KEEP/SCALE |
| D2 | ↑ | ↓ | Margin risk | PRICE TEST |
| D3 | ↓ | ↑ | Promo success | SCALE PROMO |
| D4 | ↓ | ↓ | Double hit | FIX/DELIST |

___

# Stage 2: LLM Reporter  <a name="stage2"></a>

<br>
### Prompt Engineering  <a name="prompts"></a>

The LLM generates structured insights:

```python
def llama_story_v2(llm, llm_input):
    system_message = (
        "You are a senior performance analyst. "
        "Write a weekly causal summary in ENGLISH.\n\n"
        "STRICT RULES:\n"
        "1) Use ONLY numbers from the input\n"
        "2) Do NOT invent metrics\n"
        "3) Output ONLY valid JSON\n"
        "4) Focus on: WHAT (ΔR), WHY (PVM), WHERE (Quadrants), ACTIONS\n"
    )

    user_message = (
        "Output JSON with these fields:\n"
        '{"headline": string, "executive_summary": [string], '
        '"story": string, "top_findings": [string], '
        '"recommended_actions": [string]}\n\n'
        "INPUT:\n" + json.dumps(payload)
    )

    resp = llm.create_chat_completion(
        messages=[
            {"role": "system", "content": system_message},
            {"role": "user", "content": user_message}
        ],
        temperature=0.0,
        max_tokens=1400
    )

    return json_loads_robust(resp["choices"][0]["message"]["content"])
```

**Prompt Techniques:**

1. **Structured Output**: Exact JSON schema
2. **Constraints**: "Use ONLY input numbers" prevents hallucination
3. **Focus**: "WHAT, WHY, WHERE, ACTIONS"
4. **Temperature**: 0.0 for reproducibility

<br>
### Robust JSON Parsing  <a name="parsing"></a>

Handle LLM output reliably:

```python
def json_loads_robust(text):
    try:
        return json.loads(text)
    except:
        # Extract first complete JSON object
        extracted = extract_first_json_object(text)
        if extracted:
            return json.loads(extracted)
        # Last resort: repair with LLM
        return repair_json_with_llm(llm, text)

def extract_first_json_object(text):
    start = text.find("{")
    if start < 0: return None

    depth = 0
    for i in range(start, len(text)):
        if text[i] == "{": depth += 1
        elif text[i] == "}":
            depth -= 1
            if depth == 0:
                return text[start:i+1]
    return None
```

___

# PVM Methodology Deep Dive  <a name="pvm-deep"></a>

**Price-Volume-Mix decomposition** requires careful universe definition:

## Universe Types

**Common Products** (Bridge):
- Has any revenue/quantity in T1 AND T2
- Ensures bridge reconciliation

**Priced Products** (PVM):
- Has non-zero quantity in T1 AND T2
- Unit prices p1, p2 computable
- Gets Volume and Price effects

**Unpriced Products**:
- Common but q1=0 or q2=0
- Goes into Mix effect

## Volume Effect

Answers: *"At old prices, what would new quantity generate?"*

```python
volume_effect = p1 * (q2 - q1)
```

**Positive Volume**: Sold more units (good)
**Negative Volume**: Sold fewer units (concerning)

## Price Effect

Answers: *"At new quantity, what did price change contribute?"*

```python
price_effect = q2 * (p2 - p1)
```

**Positive Price**: Higher prices (margin opportunity)
**Negative Price**: Lower prices (promotional/competitive)

## Mix Effect

Residual capturing:
- Product mix shifts
- Unpriced product changes
- Cross-effects

```python
mix_effect = delta_common - volume - price
```

Typically 5-15% of total ΔR.

___

# Quadrant Logic & Actions  <a name="quadrant-logic"></a>

## D1: Price↑ Volume↑ (Growth Zone)

**Meaning**: More units at higher prices
**Cause**: Strong product-market fit
**Action**: KEEP / PRICE-UP / SCALE

```python
if pe > 0 and ve > 0:
    if delta_margin > 0:
        return "KEEP / PRICE-UP"
    return "KEEP (watch margin)"
```

## D2: Price↑ Volume↓ (Margin Risk)

**Meaning**: Higher price, fewer units
**Cause**: Aggressive pricing or competitor pressure
**Action**: PRICE DOWN TEST

Warning: If margin is negative, price increase backfired.

## D3: Price↓ Volume↑ (Promotional)

**Meaning**: Lower price driving volume
**Cause**: Active promotions
**Action**: PROMO SCALE (if margin positive) or PROMO STOP (if negative)

## D4: Price↓ Volume↓ (Double Hit)

**Meaning**: Lower price AND fewer units
**Cause**: Product becoming obsolete or availability issues
**Action**: FIX AVAILABILITY or DELIST

```python
if pe < 0 and ve < 0:
    if in_stock_flag == 0:
        return "FIX AVAILABILITY"
    return "DELIST"
```

___

# Example Output  <a name="results"></a>

Running on Nov 2025 vs Dec 2025 data:

```
Weekly Causal Summary

Periods: t1 = 2025-11-01 – 2025-11-30 | t2 = 2025-12-01 – 2025-12-31

Summary
 - Total revenue: t1 = 204,382,935 Ft
                  t2 = 217,341,392 Ft
                  → Δ = 12,958,457 Ft

Executive Summary
 Revenue grew by 13M Ft driven by common products (+12M). Volume effect
 dominated at +14M Ft (order count +917), offset by Price (-1.4M) and Mix
 (-0.3M). Growth zone (D1) led with 153 products. Repeat buyers contributed
 87% of volume growth. Top product: Biofinity (6db) at +3.1M Ft.

Revenue Bridge
 - Appearing (t2-only): 35,511,503 Ft
 - Disappearing (t1-only): 34,924,704 Ft
 - Common products Δ: 12,371,659 Ft
 - Total Δ: 12,958,458 Ft

PVM Decomposition
 - ΔR (common): 12,371,659 Ft
 - Volume effect: 14,094,044 Ft  [(q2−q1) × p1]
 - Price effect: -1,383,767 Ft  [q2 × (p2−p1)]
 - Mix effect: -338,617 Ft      [residual]

Volume Breakdown
 - Order count: t1=12,252, t2=13,169, Δ=917
 - Basket size: t1=2.704, t2=2.689, Δ=-0.015
 - Interpretation: order-driven growth

 - Repeat buyers: 86.8% of volume effect
   Top products: Avaira Vitality (6db) +5M, Biofinity (6db) +3M

 - One-time buyers: 13.2% of volume effect
   Top products: RB_4165_60171 +468K

Quadrant Analysis
 - D1 (Growth Zone): 153 products, +12.9M Ft impact
   Top: Biofinity (6db) +3.1M
   Action: KEEP / PRICE-UP

 - D2 (Margin Risk): 124 products, -7.0M Ft impact
   Top: Biofinity (3db) -1.0M
   Action: PRICE DOWN TEST

 - D3 (Promotional): 122 products, +12.5M Ft impact
   Top: Avaira Vitality (6db) +3.8M
   Action: PROMO SCALE

 - D4 (Double Hit): 94 products, -5.7M Ft impact
   Top: Dailies Total 1 (90db) -1.3M
   Action: DELIST / FIX AVAILABILITY
```

**Key Insights:**

1. Volume drove growth (+14M), Price slightly negative
2. Repeat buyers = 87% of volume growth
3. D1 (growth zone) had most products and impact
4. Top single product: Biofinity 6-pack (+3.1M Ft)

___

# Configuring English Output  <a name="english-config"></a>

The system was originally Hungarian. To produce English:

**Step 1: Update LLM prompts**

In `llm_v2.py`, modify:

```python
system_message = (
    "You are a senior performance analyst. "
    "Write a weekly causal summary in ENGLISH.\n"  # ← Add this
    # ...
)
```

**Step 2: Translate static text**

In `render_human_report()`:

```python
lines.append("Weekly Causal Summary")     # was: "Heti összefoglaló"
lines.append("Summary")                   # was: "Összefoglaló"
lines.append("Revenue Bridge")            # already English
lines.append("Volume effect explanation") # was: "Volume-hatás magyarázat"
```

**Step 3: Update column headers**

```python
headers = ["Product", "Price", "Volume", "Impact", "Margin Δ", "Action"]
# was: ["CT2_pack", "Ár", "Mennyiség", "Impact", "Margin Δ", "Akció"]
```

**Result:** Native English output without manual translation.

___

# Next Steps  <a name="growth-next-steps"></a>

**Current Limitations:**

1. **Two-period only**: No trend analysis
2. **Manual date selection**: User specifies T1/T2
3. **Static inventory**: Snapshot only
4. **Text output**: No visualizations

**Planned Enhancements:**

1. **Multi-Period Tracking**
   - Store historical results in database
   - Trend detection: "3rd month of D2 decline"
   - Seasonality analysis

2. **Automated Period Detection**
   - Auto-select recent periods
   - Support WoW, MoM, YoY comparisons

3. **Inventory Time-Series**
   - Daily snapshots
   - Availability impact: "Out of stock 12 days in T2"
   - Forecast integration

4. **Visualization Layer**
   - Waterfall charts (PVM)
   - Quadrant scatter plots
   - Product performance heatmaps

5. **Action Tracking**
   - Log recommendations with timestamps
   - Track implementation and measure impact
   - Close the feedback loop

6. **Interactive Dashboard**
   - Streamlit UI for business users
   - Drill-down capabilities
   - Export to PowerPoint/PDF

7. **Alerting System**
   - Email for D4 products above threshold
   - Slack for significant PVM swings
   - Proactive weekly summaries

This project demonstrates how **deterministic analytics** (PVM) combined with **LLM reasoning** can automate complex business intelligence while maintaining auditability.
