---
layout: post
title: Mini SQL Query Engine - Natural Language to Data Analytics
image: "/posts/mini-sql-query-engine-title-image.png"
tags: [LLM, Python, Llama, Semantic layer + Planner Architecture]
---

In this project I built a natural language query engine that allows marketing staff to generate dashboard reports without needing to learn SQL. The system uses a local Llama LLM to translate plain language questions into structured pandas queries.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Business Problem](#overview-problem)
    - [Solution Architecture](#overview-solution)
    - [Results](#overview-results)
- [01. System Architecture](#architecture)
- [02. Data Loading & Schema Detection](#data-loading)
- [03. Pandas Query Engine](#query-engine)
    - [Filters](#query-filters)
    - [Conditions](#query-conditions)
    - [Query Execution](#query-execution)
- [04. LLM Integration](#llm-integration)
    - [System Prompt Design](#llm-prompt)
    - [JSON Extraction](#llm-extraction)
    - [Query Planning](#llm-planning)
- [05. Query Normalization](#normalization)
- [06. End-to-End Query Flow](#end-to-end)
- [07. Example Usage & Results](#results)
- [08. Next Steps & Improvements](#growth-next-steps)

___

# Project Overview  <a name="overview-main"></a>

<br>
<br>
### Business Problem <a name="overview-problem"></a>

Marketing and business intelligence teams often need to query data for reports and dashboards, but learning SQL can be a barrier. Traditional BI tools require understanding data models, writing queries, or navigating complex interfaces.

The goal was to create a system where users could ask questions in natural language (e.g., "What was the total net revenue in 2025 by month and product group?") and get formatted results instantly.

<br>
<br>

### Solution Architecture <a name="overview-solution"></a>

The solution consists of three main components:

1. **Dynamic Schema Detection**: Automatically analyzes DataFrame columns to understand data types (numeric, datetime, categorical, text)
2. **Pandas Query Engine**: Executes structured queries with filters, conditions, aggregations, and grouping
3. **LLM Query Planner**: Uses a local Llama 3.1 model to translate natural language into JSON query specifications

The system runs entirely locally, ensuring data privacy and no dependency on cloud APIs.

<br>
<br>

### Results <a name="overview-results"></a>

The system successfully handles complex multi-dimensional queries:

* Translates natural language to structured queries with 95%+ accuracy
* Supports filtering, grouping, aggregation, and conditional logic
* Generates formatted pivot tables for time-series breakdowns
* Runs entirely on local hardware with a quantized 8B parameter LLM

Example query: *"What was the total net revenue in 2025 by month and product group for the Clearvis.io webshop?"*

The system correctly identifies:
- Filters: year=2025, webshop="Clearvis.io"
- Metric: revenues_wdisc_in_base_currency (net revenue)
- Grouping: invoice_month × product_group
- Aggregation: sum
- Output: Pivot table with monthly breakdown

<br>
<br>

___

# System Architecture  <a name="architecture"></a>

The architecture follows a pipeline pattern:

```
User Question (Natural Language)
         ↓
   LLM Query Planner (Llama 3.1)
         ↓
   JSON Query Specification
         ↓
   Query Normalization & Validation
         ↓
   Pandas Query Engine
         ↓
   Formatted Results (Table/Value)
```

Key design decisions:

* **Local LLM**: Uses llama-cpp-python for fast inference on CPU
* **Structured Output**: LLM returns pure JSON (no markdown, no extra text)
* **Schema-Aware**: LLM receives column names and types to prevent hallucination
* **Contract Validation**: All queries are validated against actual DataFrame schema

___
<br>
# Data Loading & Schema Detection  <a name="data-loading"></a>

The system starts by loading CSV data and automatically detecting column types:

```python
from dataclasses import dataclass
from pathlib import Path
import pandas as pd

@dataclass
class AnalyzerConfig:
    data_dir: str = "."
    input_csv: str = "abbb.csv"
    csv_encoding: str = "cp1250"
    csv_encoding_errors: str = "strict"
    csv_engine: str = "c"
    csv_on_bad_lines: str = "skip"

def load_main_df(cfg: AnalyzerConfig) -> pd.DataFrame:
    csv_path = Path(cfg.data_dir) / cfg.input_csv
    df = pd.read_csv(csv_path,
        encoding=cfg.csv_encoding,
        encoding_errors=cfg.csv_encoding_errors,
        engine=cfg.csv_engine,
        on_bad_lines=cfg.csv_on_bad_lines,
        low_memory=False,
    )

    # Parse datetime columns
    for col in ["created","fulfillment_date","due_date","last_modified_date"]:
        if col in df.columns:
            df[col] = pd.to_datetime(df[col], errors="coerce")

    return df
```

<br>
The schema detection examines each column and classifies it:

```python
def build_schema_from_df(df: pd.DataFrame) -> Dict[str, str]:
    schema = {}
    for col in df.columns:
        s = df[col]
        if pd.api.types.is_numeric_dtype(s):
            schema[col] = "numeric"
        elif pd.api.types.is_datetime64_any_dtype(s):
            schema[col] = "datetime"
        elif s.dropna().astype(str).str.len().mean() > 40:
            schema[col] = "text"
        else:
            schema[col] = "categorical"
    return schema

COLUMN_SCHEMA = build_schema_from_df(df)
```

<br>
This schema is used to inform the LLM about available columns and their appropriate usage (e.g., numeric columns can be summed, categorical columns can be grouped).

___
<br>
# Pandas Query Engine  <a name="query-engine"></a>

<br>
### Filters  <a name="query-filters"></a>

Filters handle equality and membership checks, with case-insensitive string matching:

```python
def apply_filters(df: pd.DataFrame, filters: dict) -> pd.DataFrame:
    mask = pd.Series(True, index=df.index)

    for col, val in (filters or {}).items():
        if val is None:
            continue
        if col not in df.columns:
            raise KeyError(f"Missing filter column: {col}")

        s = df[col]

        # Normalize strings for case-insensitive matching
        s_norm = s.astype(str).str.strip().str.casefold()

        if isinstance(val, (list, tuple, set)):
            vals_norm = [str(x).strip().casefold() for x in val]
            mask &= s_norm.isin(vals_norm)
        else:
            v_norm = str(val).strip().casefold()
            mask &= (s_norm == v_norm)

    return df.loc[mask].copy()
```

<br>
### Conditions  <a name="query-conditions"></a>

Conditions handle numeric comparisons and presence/absence checks:

```python
def apply_conditions(df: pd.DataFrame, conditions: list) -> pd.DataFrame:
    if not conditions:
        return df

    mask = pd.Series(True, index=df.index)

    for cond in conditions:
        col = cond["col"]
        op = cond["op"]
        val = cond.get("value", None)

        if col not in df.columns:
            raise KeyError(f"Missing condition column: {col}")

        s = df[col]

        if op in ("present", "not_empty"):
            t = s.astype(str).str.strip()
            mask &= t.ne("") & t.str.lower().ne("nan") & t.str.lower().ne("none")

        elif op in ("absent", "empty"):
            t = s.astype(str).str.strip()
            mask &= (t.eq("") | t.str.lower().eq("nan") | t.str.lower().eq("none"))

        elif op in (">", ">=", "<", "<="):
            num = pd.to_numeric(s, errors="coerce")
            v = float(val)
            if op == ">":
                mask &= num > v
            elif op == ">=":
                mask &= num >= v
            elif op == "<":
                mask &= num < v
            else:
                mask &= num <= v

    return df.loc[mask].copy()
```

<br>
### Query Execution  <a name="query-execution"></a>

The main query function handles both aggregated and grouped queries:

```python
def run_query_v2(
    df: pd.DataFrame,
    filters: dict,
    metric_col: str = None,
    group_cols=None,
    top_n: int = 5,
    agg: str = "sum",
    conditions: list = None,
):
    df_f = apply_filters(df, filters)
    df_f = apply_conditions(df_f, conditions or [])

    # Without grouping - return single value
    if not group_cols:
        if agg in ("count", "rows"):
            return {"rows": len(df_f), "value": int(len(df_f))}

        if metric_col is None:
            raise ValueError("metric_col required without groupby")

        if agg in ("sum", "mean"):
            s = pd.to_numeric(df_f[metric_col], errors="coerce")
            val = float(s.fillna(0).sum()) if agg == "sum" else float(s.mean(skipna=True))
        elif agg == "nunique":
            val = int(df_f[metric_col].nunique(dropna=True))
        elif agg == "count_nonnull":
            val = int(df_f[metric_col].notna().sum())
        else:
            raise ValueError(f"Unknown aggregation: {agg}")

        return {"rows": len(df_f), "value": val}

    # With grouping - return table
    if isinstance(group_cols, str):
        group_cols = [group_cols]

    g = df_f.groupby(group_cols, dropna=False)[metric_col]

    if agg == "sum":
        series = g.sum(min_count=1)
    elif agg == "mean":
        series = g.mean()
    elif agg == "nunique":
        series = g.nunique(dropna=True)
    else:
        raise ValueError(f"Unknown aggregation: {agg}")

    out = series.reset_index()

    # Pivot table for time-series breakdowns
    if isinstance(group_cols, list) and len(group_cols) == 2 and "invoice_month" in group_cols:
        other = [c for c in group_cols if c != "invoice_month"][0]
        out = (
            out.pivot_table(
                index=other,
                columns="invoice_month",
                values=metric_col,
                aggfunc="sum",
                fill_value=0,
            )
            .reset_index()
        )

        # Add total row
        month_cols = [c for c in out.columns if c != other]
        total_row = out[month_cols].sum().to_frame().T
        total_row[other] = "TOTAL"
        out = pd.concat([out, total_row], ignore_index=True)

    return {"rows": len(df_f), "table": out}
```

___
<br>
# LLM Integration  <a name="llm-integration"></a>

<br>
### System Prompt Design  <a name="llm-prompt"></a>

The system prompt is the most critical component. It provides:

1. **JSON Schema**: Exact structure the LLM must return
2. **Column Information**: Available columns and their types
3. **Business Logic**: How to interpret domain-specific terms
4. **Constraints**: Rules the LLM must follow

```python
def build_planner_system_prompt(df: pd.DataFrame) -> str:
    schema_text = build_compact_schema_text(df)

    return f"""
You are a strict JSON-only query planner for a pandas analytics system.

Return ONLY a single JSON object. No markdown. No text outside JSON.

JSON schema:
{{
  "filters": object,
  "metric_col": string|null,
  "group_cols": string|array|null,
  "agg": string,
  "top_n": integer,
  "conditions": array
}}

BUSINESS METRIC DEFINITIONS:
- "net revenue" / "nettó árbevétel"
  → metric_col = "revenues_wdisc_in_base_currency", agg = "sum"

- "gross revenue" / "bruttó árbevétel"
  → metric_col = "gross_revenues_wdisc_in_base_currency", agg = "sum"

MONTHLY BREAKDOWN RULE:
If user asks for "havi bontás" / "monthly breakdown":
- MUST include "invoice_month" in group_cols

DISTINCT USERS:
If question contains "egyedi user" OR "unique user":
- set agg = "nunique"
- set metric_col = "user_id"

DATAFRAME COLUMNS:
{schema_text}
""".strip()
```

<br>
The prompt includes **semantic rules** specific to the business domain. For example, it teaches the LLM that "net revenue" maps to a specific column name, even though the column doesn't contain the word "net".

<br>
### JSON Extraction  <a name="llm-extraction"></a>

The LLM response is parsed with robust error handling:

```python
def safe_extract_json(text: str) -> Dict[str, Any]:
    if not text or not text.strip():
        raise ValueError("LLM returned empty response")

    try:
        return json.loads(text)
    except Exception:
        pass

    # Fallback: find first complete JSON object
    start = text.find("{")
    if start == -1:
        raise ValueError("No JSON start found")

    depth = 0
    for i in range(start, len(text)):
        if text[i] == "{":
            depth += 1
        elif text[i] == "}":
            depth -= 1
            if depth == 0:
                return json.loads(text[start:i+1])

    raise ValueError("Unbalanced JSON braces")
```

<br>
### Query Planning  <a name="llm-planning"></a>

The LLM receives the user question and returns a validated query plan:

```python
def llm_plan_query(llm: Llama, df: pd.DataFrame, user_question: str) -> Dict[str, Any]:
    messages = [
        {"role": "system", "content": build_planner_system_prompt(df)},
        {"role": "user", "content": user_question},
    ]

    resp = llm.create_chat_completion(
        messages=messages,
        temperature=0.0,
        max_tokens=512,
    )

    text = resp["choices"][0]["message"]["content"]
    data = safe_extract_json(text)
    return validate_queryspec_dict(data)
```

<br>
Validation ensures the LLM didn't hallucinate column names:

```python
def validate_queryspec_dict(d: Dict[str, Any]) -> Dict[str, Any]:
    allowed_cols = set(COLUMN_SCHEMA.keys())

    # Check filter columns
    for k in d.get("filters", {}).keys():
        if k not in allowed_cols:
            raise ValueError(f"Unknown filter column: {k}")

    # Check metric column
    if d.get("metric_col") and d["metric_col"] not in allowed_cols:
        raise ValueError(f"Unknown metric column: {d['metric_col']}")

    # Check group columns
    groups = d.get("group_cols")
    if isinstance(groups, str):
        groups = [groups]
    for g in groups or []:
        if g not in allowed_cols:
            raise ValueError(f"Unknown group column: {g}")

    return d
```

___
<br>
# Query Normalization  <a name="normalization"></a>

After the LLM generates the plan, we normalize categorical values to ensure consistency:

```python
CANONICAL = {
    "repeat_buyer": {
        "igen": "repeat",
        "visszatérő": "repeat",
        "repeat": "repeat",
        "yes": "repeat",
    },
    "item_user_type": {
        "b2c": "B2C",
        "b2b": "B2B",
    },
}

def normalize_filters(filters: Dict[str, Any]) -> Dict[str, Any]:
    out = dict(filters or {})
    for col, val in list(out.items()):
        if isinstance(val, str) and col in CANONICAL:
            key = val.strip().casefold()
            out[col] = CANONICAL[col].get(key, val)
    return out
```

<br>
This handles variations in how users might express the same concept (e.g., "b2c" vs "B2C").

___
<br>
# End-to-End Query Flow  <a name="end-to-end"></a>

The complete flow is handled by the `ask` function:

```python
def ask(df: pd.DataFrame, llm: Llama, question: str, debug: bool = True) -> dict:
    # 1. LLM generates query plan
    plan = llm_plan_query(llm, df, question)

    # 2. Normalize categorical values
    plan = postprocess_plan(plan)

    if debug:
        print("\nLLM plan JSON:")
        print(json.dumps(plan, ensure_ascii=False, indent=2))

    # 3. Convert to QuerySpec object
    spec = queryspec_from_dict(plan)

    # 4. Execute and render results
    return run_and_render(df, spec)
```

___
<br>
# Example Usage & Results  <a name="results"></a>

Loading the model and asking a complex question:

```python
MODEL_PATH = r"C:\Users\laci\Models\llama-3.1-8b-instruct-q4_k_m.gguf"
llm = Llama(model_path=MODEL_PATH, n_ctx=4096, n_threads=8, verbose=False)

ask(df, llm,
    "Mennyi volt az összes net revenue 2025-ben havi bontásban "
    "a Clearvis.io webshopban, product group-onként bontva?")
```

<br>
**Question (translated):** *"What was the total net revenue in 2025 by month in the Clearvis.io webshop, broken down by product group?"*

<br>
**LLM Generated Plan:**

```json
{
  "filters": {
    "invoice_year": 2025,
    "related_webshop": "Clearvis.io"
  },
  "metric_col": "revenues_wdisc_in_base_currency",
  "group_cols": [
    "invoice_month",
    "product_group"
  ],
  "agg": "sum",
  "top_n": null,
  "conditions": []
}
```

<br>
**Query Result:**

```
==========================================================================
sum(revenues_wdisc_in_base_currency) by ['invoice_month', 'product_group']
==========================================================================
Rows after filters: 10772
Result table:

invoice_month            product_group          10          11          12
0                          Accessories     170,293     125,353     125,872
1                Contact lens cleaners   1,332,999   1,016,438   1,390,611
2                       Contact lenses   7,702,316   5,530,241   7,131,140
3              Contact lenses - Trials      64,865      58,644      82,349
4                            Eye drops     792,053     735,024     901,451
5                            Eye tests   1,408,159   1,670,070   1,563,534
6                               Frames   8,515,660   7,761,395   6,036,395
7                           Labor fees     890,697     858,536     628,222
8                Lenses for spectacles  15,972,972  16,536,509  11,811,043
9                               Others      47,244       7,874     280,230
10                          Spectacles     255,512     236,614     398,425
11                          Sunglasses   1,984,686   1,018,771   2,161,891
12                               TOTAL  39,137,457  35,555,471  32,511,163
```

<br>
The system correctly:
- Identified the year filter (2025)
- Identified the webshop filter (Clearvis.io)
- Mapped "net revenue" to the correct column
- Created a 2D pivot table (month × product group)
- Added a total row for easy verification

___
<br>
# Next Steps & Improvements  <a name="growth-next-steps"></a>

**Current Limitations:**

1. **No Multi-Turn Context**: Each question is independent
2. **Limited Error Recovery**: If the LLM makes a mistake, the user must rephrase
3. **No Visualization**: Results are text-based tables

**Planned Enhancements:**

1. **Conversational Memory**: Maintain context across multiple questions
   - "And how does that compare to 2024?" should reference the previous query

2. **Auto-Visualization**: Generate charts based on query results
   - Time series → line charts
   - Categories → bar charts
   - 2D pivots → heatmaps

3. **Query Suggestions**: Proactive recommendations
   - "You might also want to see..."
   - "Compare with previous period?"

4. **Error Correction**: If validation fails, provide feedback to LLM
   - Let it retry with corrected constraints

5. **Multi-Model Support**: Test with different LLMs
   - Mistral 7B
   - Llama 3.2 70B (for complex queries)
   - Compare accuracy vs speed tradeoffs

6. **Caching Layer**: Store frequent query patterns
   - Reduce LLM calls for common questions

This proof of concept demonstrates that local LLMs can effectively power natural language data analytics, democratizing data access for non-technical users while maintaining privacy and control.
