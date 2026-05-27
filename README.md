# M&A Deal Structuring Assistant

A multi-agent system that automates the analytical workflow a junior investment banker performs when evaluating a merger or acquisition: pro forma financial modeling, accretion/dilution analysis, capital structure optimization, stakeholder review, regulatory mapping, and negotiation strategy.

Built with LangGraph and Pydantic. Pure deterministic finance logic in this version — every number is reproducible and the math is defensible end-to-end.

## What it does

The system takes basic inputs about an acquirer and a target (revenue, net income, shares outstanding, share price, proposed deal size, financing mix) and produces a deal-structuring memo covering:

- Pro forma EPS and accretion/dilution result
- Optimal cash/stock financing mix, swept across scenarios
- Estimated regulatory timeline and required approvals
- Stakeholder retention priorities and shareholder conflicts
- Recommended negotiation posture and fallback options

The output is the kind of one-page summary a junior banker would prepare for an MD ahead of a pitch.

## The math

### Pro forma net income

```
PF_NI = Target_NI + Acquirer_NI
      + Synergies × (1 − tax)
      − (New_Debt × Cost_of_Debt) × (1 − tax)
```

Cash consideration funds new debt, which creates after-tax interest expense and reduces NI. Stock consideration creates new shares, diluting the denominator.

### Pro forma EPS

```
New_Shares = (Deal_Price × Stock%) / Acquirer_Share_Price
PF_Shares  = Acquirer_Shares + New_Shares
PF_EPS     = PF_NI / PF_Shares
```

### Accretion / dilution

```
Standalone_EPS = Acquirer_NI / Acquirer_Shares
A/D %          = (PF_EPS / Standalone_EPS) − 1
```

Result labeled Accretive (≥ 0) or Dilutive (< 0), with magnitude buckets at ±3% (Marginally), ±10% (Modestly), and beyond (Strongly).

### Capital structure optimization

For each cash percentage from 0 to 100 in 10% increments, the system recomputes pro forma EPS using the formulas above and selects the mix that maximizes EPS. This is the deterministic version of the Excel data table a banker would build to evaluate financing scenarios.

## Architecture

Six agents operate sequentially on a shared `DealState` Pydantic object, orchestrated by LangGraph:

```
Financial Model  →  Accretion/Dilution  →  Capital Structure
                                                  ↓
   Negotiation   ←   Regulatory Review   ←   Stakeholders
```

Each agent reads from the shared state, performs its analysis, and writes its outputs back. The data flow is explicit and inspectable — every intermediate value is preserved on the state object and visible after the run completes.

| Agent | Responsibility |
|---|---|
| Financial Model | Pro forma income statement, financing breakdown, share count |
| Accretion/Dilution | Pro forma EPS vs. standalone EPS, result and magnitude |
| Capital Structure | Cash/stock mix sweep to maximize EPS; timeline scaled to deal complexity |
| Stakeholders | Top management retention list, shareholder preference conflicts, incentive risk |
| Regulatory | Approvals required (HSR, EU Merger Regulation, CFIUS), antitrust exposure scaled to industry concentration |
| Negotiation | Strategy posture from acquirer priorities; fallback option from financial state |

## Quickstart

```bash
pip install langgraph pydantic
jupyter notebook Deal_Structuring_AI_Assistant_v2.ipynb
```

Run cells top to bottom. The final cell produces a formatted summary for the sample deal — a mid-cap tech acquisition with a $5B-revenue acquirer purchasing a $1B-revenue target for $2B.

## Sample output

```
════════════════════════════════════════════════════════════════
              M&A DEAL STRUCTURING SUMMARY
════════════════════════════════════════════════════════════════

DEAL OVERVIEW
  Combined Revenue:            $6.00B
  Deal Price:                  $2.00B
  Annual Synergies (pretax):   $50.0M

FINANCIAL IMPACT  (Proposed: 50% Cash / 50% Stock)
  Standalone EPS:              $5.00
  Pro Forma EPS:               $4.83
  Accretion / Dilution:        -3.3%  (Modestly Dilutive)
  New Debt Raised:             $1.00B
  New Shares Issued:           20.0M

RECOMMENDED STRUCTURE
  Optimal Mix:                 100% Cash / 0% Stock
  Expected Pro Forma EPS:      $5.42
  Expected Accretion:          +8.5%
  Estimated Timeline:          4 months

STAKEHOLDERS
  Key Management to Retain:    CEO, CFO, CTO
  Incentive Risk:              Low — multiple retention vehicles available
  Potential Conflicts:         Mixed shareholder payout preferences (cash vs. stock)

REGULATORY
  Antitrust Risk:              Elevated — concentrated industry exposure (Technology)
  Approvals Required:          HSR filing (DOJ Antitrust Division / FTC)
  Cross-Border:                No

NEGOTIATION
  Strategy:                    Collaborative — prioritize speed-to-close, accept moderate concessions
  Fallback Option:             Add contingent value rights (CVRs) or earnouts to bridge valuation gap
════════════════════════════════════════════════════════════════
```

With the sample inputs (5% pretax cost of debt, 25% effective tax rate, 20× implied P/E on equity consideration), the model correctly identifies that an all-cash structure dominates: cheap debt plus reasonable target earnings means the EPS pickup from incremental net income outweighs the after-tax interest drag, while every dollar of stock consideration would issue ~20% more shares than the implied earnings yield justifies.

## Customization

The `sample_deal` dict in the demo cell accepts any acquirer/target pair. Required inputs:

- **Target:** revenue, net income
- **Acquirer:** revenue, net income, shares outstanding, share price
- **Deal terms:** total price, proposed cash/stock split
- **Assumptions:** pretax cost of debt, effective tax rate, expected annual synergies
- **Qualitative:** management roster, shareholder preferences, industries involved, countries involved, acquirer priorities, available liquidity

## Roadmap

This deterministic version is the foundation. A research extension layers LLM reasoning onto the qualitative agents:

- **Regulatory agent** — jurisdiction-specific commentary on filing risk and likely review timelines, drawing on antitrust precedent
- **Negotiation agent** — tailored strategy narratives that condition on the full deal context rather than picking from templates
- **Stakeholder agent** — drafted retention proposals and shareholder communications

The deterministic agents remain the source of truth for any quantitative output. LLM reasoning is layered onto the narrative-heavy stages where structured judgment beats fixed rules.

## Author

Mario Yanez — Finance, Fresno State University (Fall 2026)
CFA Level 1 candidate · Honors thesis on AI applications in M&A
