# Sales Forecasting Engine

**Candidate #36** — An open-source, CRM-integrated sales forecasting platform using AI-driven deal health scoring, conversation signal extraction, and interpretable ML to predict pipeline conversion.

## Market Opportunity

The AI-in-sales market is valued at **$39.4 billion in 2025**, projected to reach **$383 billion by 2034 at 28.8% CAGR**. The sales forecasting segment (a subset) is **~22% of the market**, or $8.6 billion growing at 27% CAGR.

**Market context:**
- Clari ($150M raised, $1.6B valuation) dominates enterprise ($80K–$200K/year)
- No open-source CRM-integrated forecasting tool exists; all capable platforms are commercial
- Conversation intelligence (Gong's moat) is priced out of reach for SMB teams ($100K+/year)
- Multi-CRM consolidation (Salesforce + HubSpot simultaneously) is available only in Aviso; underserved pain point

## What This Platform Solves

An **open-source, AI-native sales forecasting engine** that integrates with Salesforce and HubSpot to score deal health, predict close probability, and explain forecast adjustments in language reps and managers trust.

**Core value proposition:**
- **Multi-CRM support**: Unified forecast across Salesforce and HubSpot simultaneously
- **Deal health scoring**: ML-driven win probability using historical closed/lost deals + activity signals
- **Interpretable AI**: Every deal score includes a one-sentence explanation of primary factors
- **Conversation signals**: Extract deal health clues from call recordings via open Whisper + LLM (low-cost Gong alternative)
- **CRM data quality scoring**: Flag stale close dates, missing activities, inflated amounts before they pollute forecasts
- **Self-hostable**: Full data sovereignty for regulated industries

## Competitive Differentiation

| Aspect | This Platform | Clari | Aviso | HubSpot | Salesforce Einstein |
|--------|---------------|-------|-------|---------|-------------------|
| **Multi-CRM** | Yes | No | Yes | No | No |
| **Conversation Signals** | Yes (Whisper+LLM) | Yes (Gong) | No | No | Yes (Einstein) |
| **Interpretable Scores** | Yes | Partial | Yes (WinScore) | No | No |
| **Open Source** | Yes | No | No | No | No |
| **Self-Hostable** | Yes | No | No | No | No |
| **Price** | Free | $80K+ | $24K–$85K | Bundled | $165+/user |

## Key Features

### Must-Have (MVP)
- OAuth 2.0 connectors for Salesforce and HubSpot
- Deal-level ML win probability scoring from historical data
- Team roll-up forecast with Commit / Best Case / Most Likely categories
- Natural-language deal score explanation (one-sentence justification)
- Slack alerting for at-risk deals (no activity in 14 days, probability drop >15 points)
- Historical forecast accuracy dashboard

### Should-Have (v1.1)
- CRM data quality scoring: flag stale close dates, missing exec contacts
- Probabilistic forecast ranges (best/worst/most-likely)
- What-if scenario modelling: adjust rep attainment and see impact
- Pipeline coverage metrics (pipeline value vs. quota)
- Multi-CRM consolidation: unified forecast across Salesforce + HubSpot

### Nice-to-Have (Backlog)
- Conversation signal extraction from call recordings (Whisper + LLM)
- Natural-language forecast query interface
- Pluggable ML backend (Prophet, XGBoost, LLM-hybrid)
- Self-hosted deployment option
- MEDDIC/MEDDPICC deal qualification scoring

## Technology Stack

**Backend**: Python, XGBoost/LightGBM for ML  
**CRM Integration**: Salesforce simple-salesforce, HubSpot API clients  
**NLP**: OpenAI Whisper (open-source model) for transcription  
**ML Explainability**: SHAP for feature importance  
**Licensing**: MIT or Apache 2.0 (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–5): Salesforce connector + HubSpot connector + ML scoring + explanations
2. **Conversation Signals** (months 6–9): Whisper transcription + LLM-based deal health extraction
3. **Enterprise Features** (months 10–14): Data quality scoring, what-if scenarios, multi-CRM analytics
4. **Monetization**: Open-source core + managed cloud tier ($500–$2K/month), enterprise support ($5K+/month)

## Why This Matters

- **No open-source alternative exists**: Clari, Aviso, and Gong are all commercial. Open-source option captures engineer-forward SMBs
- **Conversation intelligence democratization**: Gong's call signal extraction is priced at $100K+/year. Open Whisper + LLM can deliver 80% of signal at 1% of cost
- **Multi-CRM gap**: Post-M&A companies often have both Salesforce and HubSpot. Only Aviso handles this; an open-source alternative would be valuable
- **Interpretability demand**: Research shows sales managers distrust black-box forecasts. AI-generated explanations directly address adoption blocker
- **Data quality blindspot**: No tool surfaces CRM hygiene issues that inflate forecasts. This is a high-ROI, low-effort differentiator

## Success Metrics

- **Year 1**: 200+ active deployments, $150K ARR from managed cloud + support
- **Year 2**: 1,000+ active deployments, $750K ARR; featured in G2 Leaders
- **Year 3**: 3,000+ active deployments, $2.5M+ ARR; adopted by 50+ Series B+ SaaS companies
