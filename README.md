# 🏭 Production Planning System

A full **S&OP (Sales & Operations Planning)** platform built from scratch — integrating demand forecasting, inventory analytics, capacity constraints, and production suggestions into a single operational system.

Answers the question most ERPs charge a fortune to solve:  
**"What should we produce, how much, and when — given real demand, real stock, and real capacity?"**

> ⚠️ This repository contains documentation and sanitized code samples only.  
> The production system runs on private infrastructure connected to a live ERP database.

---

## 📸 Screenshots

### Production Planning Dashboard
![Production Planning Dashboard](screenshots/Production%20Planning%20Dashboard.png)

### Production Plan Suggestion
![Production Plan Suggestion](screenshots/Sugestão%20Plano.png)

### ABC Curve Analysis
![ABC Curve](screenshots/Production%20Planning%20ABC.png)

---

## 🧠 Full S&OP Pipeline

This is not a simple MRP. It is a complete demand-supply planning cycle:

```
Step 1 — Historical Sales (828,613 records)
         │
         ▼
Step 2 — Smart Minimum Stock
         3 business rules applied per SKU
         based on sales trend (growth · stable · decline)
         │
         ▼
Step 3 — ML Demand Forecasting
         XGBoost models per ABC curve
         Recursive multi-month projection per SKU
         │
         ▼
Step 4 — Available Stock Calculation
         Physical stock + In-process − Already sold
         Real-time from ERP database
         │
         ▼
Step 5 — Coverage Analysis
         How many days/weeks current available stock
         covers projected demand per SKU
         │
         ▼
Step 6 — Production Suggestion
         What to produce · How much · Priority
         Constrained by capacity configuration
         Aligned with ideal coverage targets
```

---

## 📁 Project Structure

```
production-planning-system/
├── python-backend/
│   ├── main.py                      # FastAPI app and routes
│   ├── services/
│   │   ├── mrp_service.py           # MRP + coverage calculation
│   │   ├── estoque_minimo.py        # Minimum stock business rules
│   │   ├── sugestao_plano.py        # Production suggestion engine
│   │   └── vendas_service.py        # Sales history queries
│   ├── ml/
│   │   ├── config.py                # Pipeline configuration + filters
│   │   ├── extract.py               # ERP extraction → Parquet
│   │   ├── features.py              # Feature engineering (no leakage)
│   │   ├── train.py                 # XGBoost training by ABC curve
│   │   ├── predict.py               # Recursive multi-month projection
│   │   ├── evaluate.py              # MAE · RMSE · MAPE · R² metrics
│   │   ├── diagnostics.py           # Visual diagnostic suite
│   │   ├── optimize.py              # Optuna hyperparameter search
│   │   └── main.py                  # Full pipeline runner
│   └── database.py                  # PostgreSQL connection pool
├── app/
│   ├── components/
│   │   ├── PlanoDeProdução/         # Main planning matrix
│   │   ├── SugestãoDePlano/         # Production suggestion view
│   │   ├── Projeções/               # ML forecast visualization
│   │   ├── CurvaABC/                # ABC curve analysis
│   │   ├── Capacidade/              # Capacity configuration
│   │   └── EstoqueMinimo/           # Minimum stock dashboard
│   └── hooks/
│       ├── usePlanejamento.ts
│       └── useProjecoes.ts
└── screenshots/
```

---

## ⚙️ Core Logic

### Step 1 — Smart Minimum Stock (3 Business Rules)

Applied over 828,613 real sales records. Each SKU gets a dynamic minimum stock based on its own sales trend.

```python
def calcular_estoque_minimo(media_semestral: float, media_trimestral: float) -> dict:
    variacao = ((media_trimestral - media_semestral) / media_semestral) * 100

    if variacao > THRESHOLD_CRESCIMENTO:
        # Rule 1 — Growth: weight toward recent demand
        estoque_minimo = (media_semestral * 0.3) + (media_trimestral * 0.7)
        regra = 1

    elif variacao < THRESHOLD_QUEDA:
        # Rule 2 — Decline: conservative, use lower average
        estoque_minimo = min(media_semestral, media_trimestral)
        regra = 2

    else:
        # Rule 3 — Stable: balanced average
        estoque_minimo = (media_semestral + media_trimestral) / 2
        regra = 3

    return {
        "estoqueMinimo": round(estoque_minimo, 2),
        "variacaoPercentual": round(variacao, 2),
        "regraAplicada": regra
    }
```

---

### Step 2 — ML Forecasting Pipeline

Segmented XGBoost models trained per ABC curve — different complexity for different demand patterns.

```python
# Segmentation strategy per curve
CURVE_MODELS = {
    "A": "xgboost_full",       # Full feature set — high-value SKUs
    "B": "xgboost_simplified", # Reduced features — mid-tier SKUs
    "C/D": "group_level"       # Group prediction + share-weighted SKU allocation
}

# Features engineered without temporal leakage
FEATURES = [
    "lag_1", "lag_2", "lag_3",           # Recent demand lags
    "rolling_avg_3m", "rolling_avg_6m",  # Rolling averages
    "trend_slope",                         # Linear trend direction
    "yoy_ratio",                           # Year-over-year ratio
    "month_sin", "month_cos"              # Cyclical seasonality encoding
]

# TimeSeriesSplit — no data leakage between train and validation
tscv = TimeSeriesSplit(n_splits=5)

# Optuna — Bayesian hyperparameter search
study = optuna.create_study(direction="minimize")  # minimize MAPE
study.optimize(objective, n_trials=100)
```

**Target metrics:** MAPE < 20% · R² > 0.70

---

### Step 3 — Available Stock Calculation

```python
def calcular_disponivel(produto: dict) -> dict:
    # Real-time from ERP
    estoque_fisico = get_estoque_fabrica(produto["idproduto"])
    em_processo = get_em_processo(produto["idproduto"])
    ja_vendido = get_pedidos_pendentes(produto["idproduto"])

    # What's actually available to cover future demand
    disponivel = estoque_fisico + em_processo - ja_vendido

    return {
        "estoque_fisico": estoque_fisico,
        "em_processo": em_processo,
        "ja_vendido": ja_vendido,
        "disponivel": max(0, disponivel)
    }
```

---

### Step 4 — Coverage Analysis

How long current available stock covers projected demand.

```python
def calcular_cobertura(disponivel: float, projecao_mensal: float) -> dict:
    if projecao_mensal <= 0:
        return {"cobertura_meses": None, "status": "SEM_DEMANDA"}

    cobertura_meses = disponivel / projecao_mensal

    if cobertura_meses >= COBERTURA_IDEAL:
        status = "ADEQUADA"
    elif cobertura_meses >= COBERTURA_MINIMA:
        status = "ATENCAO"
    else:
        status = "CRITICA"

    return {
        "cobertura_meses": round(cobertura_meses, 2),
        "cobertura_ideal": COBERTURA_IDEAL,
        "gap_cobertura": max(0, COBERTURA_IDEAL - cobertura_meses),
        "status": status
    }
```

---

### Step 5 — Production Suggestion Engine

Combines everything: minimum stock, ML forecast, available stock, coverage gap, and capacity constraints.

```python
def gerar_sugestao_plano(produto: dict, configuracoes: dict) -> dict:
    disponivel = calcular_disponivel(produto)["disponivel"]
    projecao = get_projecao_ml(produto["idproduto"])
    estoque_minimo = calcular_estoque_minimo(
        produto["media_semestral"],
        produto["media_trimestral"]
    )["estoqueMinimo"]

    # How much is needed to reach ideal coverage
    necessidade_cobertura = projecao * configuracoes["cobertura_ideal_meses"]

    # Total need = coverage target + minimum stock buffer
    necessidade_total = necessidade_cobertura + estoque_minimo

    # What still needs to be produced
    sugestao_bruta = max(0, necessidade_total - disponivel)

    # Apply capacity constraint
    capacidade_disponivel = configuracoes["capacidade_maxima_periodo"]
    sugestao_final = min(sugestao_bruta, capacidade_disponivel)

    return {
        "sugestao_producao": round(sugestao_final, 0),
        "necessidade_total": round(necessidade_total, 0),
        "disponivel": round(disponivel, 0),
        "cobertura_atual": round(disponivel / projecao, 2) if projecao > 0 else None,
        "cobertura_apos_producao": round((disponivel + sugestao_final) / projecao, 2),
        "limitado_por_capacidade": sugestao_final < sugestao_bruta,
        "prioridade": classificar_prioridade(disponivel, estoque_minimo, sugestao_final)
    }
```

---

## 🔌 API Reference

```
GET  /api/producao/planejamento                     # Bulk planning matrix (up to 200 SKUs)
GET  /api/producao/planejamento/:cdProduto          # Single product full analysis
GET  /api/producao/sugestao-plano                   # Production suggestions with capacity
GET  /api/producao/estoque                          # Current physical stock
GET  /api/producao/em-processo                      # In-process production
GET  /api/producao/pedidos-pendentes/:cdProduto     # Pending orders
GET  /api/producao/catalogo                         # Product catalog with filters
GET  /api/producao/cobertura                        # Coverage analysis per SKU
POST /api/ml/run-pipeline                           # Trigger ML forecast pipeline
GET  /api/vendas/produtos/:id/estoque-minimo        # Minimum stock per SKU
GET  /api/vendas/produtos/:id/estatisticas          # Full sales statistics
```

---

## 📊 ML Diagnostic Output

| Output | Description |
|---|---|
| `regression_curve_*.png` | Predicted vs actual scatter with identity line |
| `anomalies_curve_*.png` | Outlier detection via IQR |
| `importance_curve_*.png` | Feature importance ranking per curve |
| `temporal_curve_*.png` | Temporal series — predicted vs actual over time |
| `diagnostic_report.txt` | Full qualitative metrics report |
| `app_ml_runs` (PostgreSQL) | Run metadata, parameters, and metrics per execution |
| `app_ml_forecasts` (PostgreSQL) | SKU-level forecasts per month |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 · TypeScript · Tailwind CSS · Recharts |
| Backend | Python · FastAPI · Uvicorn |
| ML | XGBoost · LightGBM · Optuna · Scikit-learn · Pandas · Parquet |
| Database | PostgreSQL · psycopg2 · Connection pooling |
| Infra | Render · Cron scheduling · Structured logging |

---

## 🔐 Notes

- All credentials managed via environment variables — never committed
- ERP database functions abstracted in service layer
- Production data, business-specific filters, and capacity configurations removed from this public version
