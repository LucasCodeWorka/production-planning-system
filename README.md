# production-planning-system# 🏭 Production Planning System

An end-to-end manufacturing intelligence platform — from sales forecasting to production order prioritization.

Integrates real ERP data across 5 sources to answer one question automatically:  
**"What do we need to produce today, and how urgent is it?"**

> ⚠️ This repository contains documentation and sanitized code samples only.  
> The production system runs on private infrastructure connected to a live ERP database.

---

## 📸 Screenshots

### Production Planning Dashboard
![Production Planning Dashboard](screenshots/dashboard.png)

### MRP Matrix — Projections by Month
![MRP Matrix](screenshots/mrp-matrix.png)

### ABC Curve Analysis
![ABC Curve](screenshots/abc-curve.png)

---

## 🧠 System Architecture

```
ERP Database (PostgreSQL)
        │
        ├── Physical Stock ──────────────────┐
        ├── In-Process Production ───────────┤
        ├── Pending Customer Orders ─────────┼──▶ MRP Engine ──▶ Priority Classification
        ├── Sales History (828k+ records) ───┤         │
        └── Product Catalog ─────────────────┘         │
                                                        ▼
                                              Smart Minimum Stock
                                              (3 Business Rules)
                                                        │
                                                        ▼
                                              ML Forecasting Pipeline
                                              (XGBoost · Optuna · Parquet)
                                                        │
                                                        ▼
                                              Next.js Dashboard
                                              (Planning · ABC · Projections)
```

---

## 📁 Project Structure

```
production-planning-system/
├── python-backend/
│   ├── main.py                    # FastAPI app and routes
│   ├── services/
│   │   ├── mrp_service.py         # MRP calculation engine
│   │   ├── estoque_minimo.py      # Minimum stock business rules
│   │   └── vendas_service.py      # Sales history queries
│   ├── ml/
│   │   ├── config.py              # Pipeline configuration
│   │   ├── extract.py             # ERP data extraction → Parquet
│   │   ├── features.py            # Feature engineering (no leakage)
│   │   ├── train.py               # XGBoost training by ABC curve
│   │   ├── predict.py             # Recursive multi-month projection
│   │   ├── evaluate.py            # MAE, RMSE, MAPE, R² metrics
│   │   ├── diagnostics.py         # Visual diagnostic suite
│   │   ├── optimize.py            # Optuna hyperparameter search
│   │   └── main.py                # Full pipeline runner
│   └── database.py                # PostgreSQL connection pool
├── app/                           # Next.js frontend
│   ├── components/
│   │   ├── PlanoDeProdução/       # Planning matrix
│   │   ├── CurvaABC/              # ABC curve analysis
│   │   ├── Projeções/             # ML forecast views
│   │   └── EstoqueMinimo/         # Minimum stock dashboard
│   └── hooks/
│       ├── usePlanejamento.ts
│       └── useProjecoes.ts
└── scripts/
    └── testar-producao.js         # Integration test runner
```

---

## ⚙️ Core Logic

### 1. Smart Minimum Stock Engine

Calculates dynamic minimum stock per SKU based on sales trend analysis over **828,613 historical records**.

```python
def calcular_estoque_minimo(media_semestral: float, media_trimestral: float) -> dict:
    variacao = ((media_trimestral - media_semestral) / media_semestral) * 100

    if variacao > THRESHOLD_CRESCIMENTO:
        # Rule 1 — Growth scenario: weight toward recent data
        estoque_minimo = (media_semestral * 0.3) + (media_trimestral * 0.7)
        regra = 1

    elif variacao < THRESHOLD_QUEDA:
        # Rule 2 — Decline scenario: conservative, use lower average
        estoque_minimo = min(media_semestral, media_trimestral)
        regra = 2

    else:
        # Rule 3 — Stable scenario: balanced average
        estoque_minimo = (media_semestral + media_trimestral) / 2
        regra = 3

    return {
        "estoqueMinimo": round(estoque_minimo, 2),
        "variacaoPercentual": round(variacao, 2),
        "regraAplicada": regra
    }
```

### 2. MRP Calculation Engine

```python
def calcular_necessidade_producao(produto: dict) -> dict:
    # Integrate all 5 data sources
    estoque_disponivel = estoque_atual + em_processo
    necessidade_total = estoque_minimo + pedidos_pendentes
    necessidade_producao = max(0, necessidade_total - estoque_disponivel)

    # Priority classification
    if estoque_atual < estoque_minimo:
        prioridade = "ALTA"    # Product already below minimum — urgent
    elif necessidade_producao > 0:
        prioridade = "MEDIA"   # Production needed, but stock still OK
    else:
        prioridade = "BAIXA"   # No action required

    return {
        "necessidade_producao": necessidade_producao,
        "prioridade": prioridade,
        "situacao": "PRODUZIR" if necessidade_producao > 0 else "ESTOQUE_OK"
    }
```

### 3. ML Forecasting Pipeline

Segmented XGBoost models trained per ABC curve — different strategies for different demand patterns.

```python
# Segmentation strategy
CURVE_MODELS = {
    "A": "xgboost_full",        # Full feature set, complex model
    "B": "xgboost_simplified",  # Reduced features, lighter model
    "C/D": "group_level"        # Group prediction + share-weighted allocation per SKU
}

# Feature engineering — no temporal leakage
FEATURES = [
    "lag_1", "lag_2", "lag_3",          # Recent lags
    "rolling_avg_3m", "rolling_avg_6m", # Rolling averages
    "trend_slope",                        # Linear trend
    "yoy_ratio",                          # Year-over-year ratio
    "month_sin", "month_cos"             # Cyclical seasonality encoding
]

# Validation — TimeSeriesSplit prevents data leakage
tscv = TimeSeriesSplit(n_splits=5)
for train_idx, val_idx in tscv.split(X):
    # Train only on past, validate on future
    ...
```

**Hyperparameter optimization with Optuna:**
```python
def objective(trial):
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 100, 500),
        "max_depth": trial.suggest_int("max_depth", 3, 8),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3),
        "subsample": trial.suggest_float("subsample", 0.6, 1.0),
    }
    # Bayesian search across XGBoost and LightGBM
    ...

study = optuna.create_study(direction="minimize")  # minimize MAPE
study.optimize(objective, n_trials=100)
```

**Target metrics:** MAPE < 20% · R² > 0.70

---

## 🔌 API Reference

```
GET  /api/producao/planejamento                    # Bulk planning (up to 200 SKUs)
GET  /api/producao/planejamento/:cdProduto         # Single product full analysis
GET  /api/producao/estoque                         # Current physical stock
GET  /api/producao/em-processo                     # In-process production
GET  /api/producao/pedidos-pendentes/:cdProduto    # Pending orders by product
GET  /api/producao/catalogo                        # Full product catalog
POST /api/ml/run-pipeline                          # Trigger ML forecast pipeline
GET  /api/vendas/produtos/:id/estoque-minimo       # Minimum stock calculation
GET  /api/vendas/produtos/:id/estatisticas         # Full sales statistics
```

**Example — products that need production, ordered by priority:**
```bash
curl "/api/producao/planejamento?apenas_necessidade=true&ordenar_por=prioridade&limit=50"
```

**Example response:**
```json
{
  "produto": { "apresentacao": "PRODUCT NAME", "continuidade": "PERMANENTE" },
  "estoques": {
    "estoque_atual": 45,
    "em_processo": 120,
    "estoque_disponivel": 165,
    "estoque_minimo": 200
  },
  "demanda": {
    "pedidos_pendentes": 80,
    "media_vendas_6m": 95.4,
    "media_vendas_3m": 110.2
  },
  "planejamento": {
    "necessidade_producao": 115,
    "prioridade": "ALTA",
    "situacao": "PRODUZIR"
  }
}
```

---

## 📊 ML Diagnostic Output

The pipeline generates a full diagnostic suite after each run:

| Output | Description |
|---|---|
| `regression_curve_*.png` | Predicted vs actual scatter with identity line |
| `anomalies_curve_*.png` | Outlier detection via IQR method |
| `importance_curve_*.png` | Feature importance ranking per curve |
| `temporal_curve_*.png` | Temporal series — predicted vs actual over time |
| `diagnostic_report.txt` | Full text report with qualitative metrics |
| `app_ml_runs` (PostgreSQL) | Run metadata, parameters, and metrics |
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
- ERP database functions (`f_dic_sld_prd_produto`) abstracted in the service layer
- Production data and business-specific filters removed from this public version
