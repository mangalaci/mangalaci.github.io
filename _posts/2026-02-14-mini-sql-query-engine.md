---
layout: post
title: "Mini SQL Query Engine — Natural Language Analytics with Local LLM"
date: 2026-02-14
image: "/posts/mini-sql-query-engine-title-image.png"
categories: [Python, LLM, Metric Registry, Semantic layer + Planner Architecture]
---  

## Overview

This project is a locally-run natural language analytics system that lets non-technical users query business data in plain language. No SQL knowledge required, no cloud dependencies — everything runs on a quantized Llama 3.1 8B model.

## Key Design Principle

The LLM's job is intentionally limited: **it only extracts intent**, it does not execute anything.

Given a natural language question, the LLM returns a small JSON object describing what the user wants — which filters to apply, which columns to group by, which metric to aggregate. The program then executes this plan deterministically using plain Python and pandas.

```
Question
  ↓
LLM → {"filters": {"invoice_year": 2025, "related_webshop": "Clearvis.io"},
        "metric_col": "revenues_wdisc_in_base_currency",
        "group_cols": ["invoice_month", "product_group"],
        "agg": "sum"}
  ↓
run_plan() — deterministic Python
  ↓
Result table
```

This separation is the core of the design. The LLM is good at understanding language; it is not reliable as a code generator. By constraining it to JSON extraction, every failure is explicit and catchable before any data is touched.

## Full Implementation

```python
import json
import pandas as pd
from llama_cpp import Llama


CSV_PATH   = r"C:\path\to\data.csv"
MODEL_PATH = r"C:\path\to\llama-3.1-8b-instruct-q4_k_m.gguf"


df = pd.read_csv(
    CSV_PATH,
    encoding="cp1250",
    encoding_errors="ignore",
    on_bad_lines="skip",
    low_memory=False,
)

llm = Llama(model_path=MODEL_PATH, n_ctx=4096, n_threads=8, verbose=False)


SYSTEM_PROMPT = """You are a query planner. Extract query parameters from the user's question.

Return ONLY a JSON object with these fields:
{
  "filters": {column: value, ...},
  "metric_col": string or null,
  "group_cols": [string, ...] or null,
  "agg": "sum" | "mean" | "count" | "nunique"
}

Column mappings (USE THESE EXACT COLUMN NAMES):
- webshop / shop      -> related_webshop   (values: "eOptika.hu", "LentileContact.ro", "Clearvis.io")
- net revenue         -> revenues_wdisc_in_base_currency
- gross revenue       -> gross_revenues_wdisc_in_base_currency
- year                -> invoice_year      (integer)
- month / havi bontás -> invoice_month     (integer, 1-12)
- product group       -> product_group
- user count          -> agg="nunique", metric_col="user_id"

Rules:
- filters: equality conditions only (column -> value)
- group_cols: always a list; include invoice_month if monthly breakdown is asked
- agg: "sum" for revenue, "count" for rows, "nunique" for unique users
- Return ONLY the JSON, no explanation

Example:
Q: What was the total net revenue in 2025 by month in the Clearvis.io webshop by product group?
A: {"filters": {"invoice_year": 2025, "related_webshop": "Clearvis.io"}, "metric_col": "revenues_wdisc_in_base_currency", "group_cols": ["invoice_month", "product_group"], "agg": "sum"}
"""


def question_to_plan(question: str) -> dict:
    resp = llm.create_chat_completion(
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user",   "content": question},
        ],
        temperature=0,
        max_tokens=256,
    )
    raw = resp["choices"][0]["message"]["content"]
    start = raw.find("{")
    end   = raw.rfind("}") + 1
    return json.loads(raw[start:end])


def run_plan(df: pd.DataFrame, plan: dict) -> pd.DataFrame:

    # filters — case-insensitive
    mask = pd.Series(True, index=df.index)
    for col, val in (plan.get("filters") or {}).items():
        if val is None or col not in df.columns:
            continue
        s_norm = df[col].astype(str).str.strip().str.casefold()
        v_norm = str(val).strip().casefold()
        mask &= (s_norm == v_norm)

    df_f = df.loc[mask]

    metric     = plan.get("metric_col")
    group_cols = plan.get("group_cols") or []
    agg        = plan.get("agg", "sum")

    if not group_cols:
        if agg == "count":
            return pd.DataFrame({"result": [len(df_f)]})
        s = pd.to_numeric(df_f[metric], errors="coerce")
        return pd.DataFrame({"result": [s.sum() if agg == "sum" else s.mean()]})

    if agg == "count":
        result = df_f.groupby(group_cols).size().reset_index(name="count")
    elif agg == "nunique":
        result = df_f.groupby(group_cols)[metric].nunique().reset_index()
    else:
        df_f = df_f.copy()
        df_f[metric] = pd.to_numeric(df_f[metric], errors="coerce")
        result = df_f.groupby(group_cols)[metric].sum().reset_index() if agg == "sum" \
            else df_f.groupby(group_cols)[metric].mean().reset_index()

    # pivot: invoice_month + 1 other dimension → wide table
    if "invoice_month" in group_cols and len(group_cols) == 2:
        other = [c for c in group_cols if c != "invoice_month"][0]
        result = result.pivot_table(
            index=other,
            columns="invoice_month",
            values=metric,
            aggfunc="sum",
            fill_value=0,
        ).reset_index()
        month_cols = sorted(
            [c for c in result.columns if c != other], key=lambda x: int(x)
        )
        result = result[[other] + month_cols]

    return result


def ask(question: str):
    print("\nQUESTION:", question)
    plan = question_to_plan(question)
    print("\nPLAN:")
    print(json.dumps(plan, ensure_ascii=False, indent=2))
    result = run_plan(df, plan)
    print("\nRESULT:")
    print(result.to_string(index=False))
    return result


ask("What was the total net revenue in 2025 by month in the Clearvis.io webshop by product group?")
```

## Example Output

```
PLAN:
{
  "filters": {"invoice_year": 2025, "related_webshop": "Clearvis.io"},
  "metric_col": "revenues_wdisc_in_base_currency",
  "group_cols": ["invoice_month", "product_group"],
  "agg": "sum"
}

Rows after filters: 10772

RESULT:
             product_group          10          11          12
               Accessories     170,293     125,353     125,872
 Contact lens cleaners       1,332,999   1,016,438   1,390,611
        Contact lenses       7,702,316   5,530,241   7,131,140
                Eye drops      792,053     735,024     901,451
               Eye tests      1,408,159   1,670,070   1,563,534
                  Frames      8,515,660   7,761,395   6,036,395
  Lenses for spectacles      15,972,972  16,536,509  11,811,043
              Sunglasses      1,984,686   1,018,771   2,161,891
```

## Why It Works

The system prompt does the heavy lifting: it maps business language to exact column names, provides example values, and gives the LLM a single concrete example to follow. At `temperature=0` the output is deterministic.

Filters use case-insensitive matching, so slight casing differences between user input and data values are handled automatically. The pivot logic kicks in automatically when a monthly breakdown is combined with a second grouping dimension.

The entire system runs locally — no API calls, no data leaving the machine.
