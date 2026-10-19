# CLAUDE.md — Project Fulcrum

Context file for Claude Code and for me (Frank). It holds the problem, the research direction and
the base-case specification the code implements. Read it in full before writing or changing code.
Mathematics uses GitHub LaTeX (`$…$`).

> **Loading note.** Claude Code only picks up this file automatically when it works on files inside
> `Docs/`. To load it in every session, put a one-line `CLAUDE.md` in the repo root containing
> `@Docs/CLAUDE.md`.

---

## 1. The project in one paragraph

Under the Dutch *Wet toekomst pensioenen* (Wtp), a pension fund credits each cohort with a
**beschermingsrendement** (protection return): a *theoretical* return, computed from changes in the
DNB interest rate term structure (RTS), that shields the cohort's expected benefit from rate moves.
The fund has to **earn** that return with real instruments (bonds, swaps, cash/MMF, repo, later
swaptions) in its interest rate overlay. What the overlay actually earns never equals the theoretical
return exactly. The gap, the **mismatch return**, comes from concrete frictions: cash drag, bond
spread risk, collateral and margin needs, curve and duration mismatches, and rebalancing. Fulcrum
treats overlay construction as a **portfolio optimisation problem over hedging instruments**, with
the mismatch return as the object being optimised. The deliverable is an **efficient frontier**:
expected mismatch against tracking error versus the theoretical protection return, under realistic
constraints.

## 2. People and setting

- **Frank Heijnen**: MSc QFAS (Quantitative Finance and Actuarial Science), Tilburg University.
  Thesis internship at **Cardano Risk Management** (Rotterdam). Also a working student in Cardano's
  collateral management team (margin-call settlement), which is a natural source for realistic
  collateral parameters.
- **Roel Mehlkopf**: company supervisor, Pension Fund Advisor, Client Advisory.
- **Marco Teunissen**: Lead Portfolio Manager LDI at Cardano. Co-author of Ernsting, Teunissen &
  Vereijken (2025), which is the practitioner statement of exactly this problem (see §3.4).
- Roel's steer, which is binding: **the method follows the problem, never the reverse.** His notes
  contain the main question: *"Optimalisatie van LDI-portefeuille voor risk-return (hoofdvraag!!!)"*,
  *"Rente overlay → theoretisch beschermingsrendement Wtp, hoe kunnen we dit het beste doen"*, and
  *"Markowitz MPT, kunnen wij hetzelfde doen voor een LDI portfolio met een set aan instrumenten
  waarmee renterisico afgedekt kan worden?"*

## 3. Institutional background

### 3.1 From FTK to Wtp
Under the old FTK regime, any deviation in realised hedging hit the fund's funding ratio
(*dekkingsgraad*) and was shared by all participants. Under the Wtp **solidaire premieregeling
(SPR)**, each cohort's capital earns two return components according to its own allocation:

- **Beschermingsrendement (BR, protection return)**: compensates for rate-driven changes in the cost
  of the cohort's pension. Protection share per cohort $\pi_c \in [0,1]$; typically low for young
  cohorts and up to 100% for retirees.
- **Overrendement (OR, excess return)**: whatever remains of the fund's total return after BR has
  been credited. Allocated by age-dependent shares, which can exceed 100% for the young (the
  *leenrestrictie*, or borrowing restriction, is lifted).

Aggregating the protection shares over cohorts gives the LDI manager a **fund-level hedging
mandate**. The LDI manager's task is unchanged from the FTK era (hedge expected cash flows) and so is
the instrument set.

### 3.2 Two ways to deliver BR
- **Praktisch / direct method**: a separate protection portfolio (bonds, swaps). Its *realised*
  return is what cohorts receive as BR.
- **Theoretisch / indirect method** (the method most funds choose): BR is **computed** from the RTS
  and credited pro rata. There is one integrated portfolio, and
  $\text{OR} = \text{total return} - \text{theoretical BR}$.

**Consequence of the indirect method.** Whatever the overlay fails to deliver (or over-delivers)
relative to the theoretical BR flows into the **OR**. Young cohorts carry the largest OR shares, so
they absorb the mismatch disproportionately. The effect is stronger in an older fund, where the
spread risk created by hedging many retirees lands on a small group of young participants
(Ernsting et al. 2025). DNB expects funds to **set, monitor and limit** this mismatch ex ante and ex
post. Tracking error against the theoretical BR is therefore a *supervised* quantity, not an
invented metric.

### 3.3 How theoretical BR is computed
BR is the percentage change in the cohort's **annuity factor** (*tarief*), i.e. the capital needed per
€1 of benefit, driven by changes in the DNB RTS. The benchmark RTS is built from the **6-month Euribor
swap curve** (Ernsting et al. 2025). It is not directly investable.

Worked monthly update (Potter van Loon & Alserda 2025, Table 1). Keep it as a unit test for the
liability module:

| Step | Tariff | Return component |
|---|---|---|
| Previous month | 10.09 | |
| Accrue required interest (short rate) | 10.12 | 0.30% |
| Age one month (micro-longevity) | 10.13 | 0.10% |
| New RTS | 10.35 | **2.17% = financial protection return** |

At protection share $\pi_c = 60\%$, the cohort is credited $0.6 \times 2.17\% = 1.30\%$ financial BR.

**Duration shortcut** (Memorie van Toelichting Wtp, pp. 36–37), with annual step, $d$ the benefit's
duration and $R_{t,\tau}$ the RTS zero rate for maturity $\tau$ at time $t$:

$$
\text{BR}_t \approx \frac{(1+R_{t,d-1})^{-(d-1)}}{(1+R_{t-1,d})^{-d}} - 1 ,
$$

which is the one-period return on a zero-coupon bond of maturity $d$. It ignores curve shape. For a
70-year-old in 2022 the tariff moved −17.99% while the shortcut gave −19.28%, so a participant who was
"100% protected" by the shortcut would still have seen the benefit fall
$(1-0.1928)/(1-0.1799)-1 = -1.57\%$. **Curve risk is material. The base case uses this shortcut and
the key-rate extension removes it.**

### 3.4 Where the mismatch comes from (Cardano's view)
Ernsting, Teunissen & Vereijken (2025) list the sources: spread results on long bonds, **cash drag**,
investment costs, curve and duration mismatches, and implementation gap.

- **Cash + swaps** tracks the RTS benchmark most closely over short horizons but earns a structural
  **cash drag**: cash, MMFs and short bonds historically yield about **25 bp/yr** below 6m Euribor.
  Roel's figure is about 20 bp below the short rate.
- **Long bonds** earn a spread over swaps that can offset the drag, but the spread fluctuates.
  Example: a 20y high-quality basket at +40 bp over swaps gives about **+8%** over 20 years, against
  about **−5%** for swaps + cash.
- The 30y FR/NL/DE government-vs-swap spread ranged from about −50 bp (around 2000) to about +90 bp
  (January 2025) and was **negative 2015–2022**, so the trade-off is state-dependent.
- Three spread policies are discussed: *no spread risk*, *spread risk sized to offset cash drag*,
  and *adaptive spread policy* (their preference). **Pull-to-par** makes spread risk less severe over
  long horizons.
- Stylised stress in the article: with 50% of the hedge in bonds (duration 15, +50 bp over swaps), a
  +50 bp spread shock gives the young cohort an OR of −2.0% in a young fund and −4.7% in an old fund.

### 3.5 Collateral and liquidity: the empirical motivation
Jansen, Klingler, Ranaldo & Duijm (2026, *RFS*), on DNB data:

- About **60%** of Dutch pension portfolio duration comes from swaps.
- Realised margin calls average **3% of AUM** in rate-surge periods and exceed **6%** for 10% of
  the sample.
- Margin calls exceed available cash buffers for **62%** of funds, and outweigh the funding-ratio
  gain from higher rates in **72%** of cases.
- Funds do **not** appear to size ex-ante cash for margin needs.
- Calls are met by selling 1–7y safe government bonds. Margin-call selling contributed **10.4 bp**
  (12%) of the 87 bp rise in the 3y euro yield in August 2022.

## 4. Research question and scope

**Main question (working version).**
> How should a pension fund construct and dynamically manage its interest rate overlay so as to
> deliver the theoretical Wtp protection return at the lowest cost and risk to participants, and
> what is the efficient frontier of that implementation problem?

**The Markowitz analogue, with different axes.**

| | Markowitz | Fulcrum |
|---|---|---|
| Choice variable | Asset weights | Instrument mix: cash / bonds / swaps (later swaptions, repo) |
| Return axis | Expected return | Expected **mismatch return** (spread earned − cash drag − costs) |
| Risk axis | Variance | **Tracking error** of the mismatch vs theoretical BR (later CVaR) |
| Constraints | Budget | Budget, hedge-ratio band, collateral/margin, spread limit, no shorting |

**Contribution: state it precisely.** Surplus optimisation (mean–variance on assets minus
liabilities) is Sharpe & Tint (1990), and later Hoevenaars et al. (2008) and Ang–Chen–Sundaresan.
**Do not claim "Markowitz with liabilities" as new.** That literature optimises over *asset classes*
and treats the hedge as a frictionless duration constraint. The gap is **one level down**: optimising
over *hedging instruments* with collateral, margin, cash drag and spread risk as first-class
frictions, measured against the **Wtp theoretical protection return**. Potter van Loon & Alserda
(2025) come closest but stop at the allocation method. Ernsting et al. (2025) state the trade-off
without optimising it.

**Candidate sub-questions** (prune to one or two pillars; the rest become robustness or further
research):
1. *Instrument choice*: optimal cash/bond/swap split for a required rate exposure; how it shifts with
   the asset-swap spread, cash drag and repo.
2. *Collateral buffer*: optimal cash held against margin calls (an inventory problem: too little
   risks forced sales, too much is drag).
3. *Convexity vs liquidity*: swaptions (premium, no margin, rising delta) versus swaps (no premium,
   margin-heavy).
4. *Rebalancing*: when correcting hedge drift pays for itself after transaction and collateral
   costs.
5. *Regimes and stress*: does the optimal overlay survive a 2022-style shock?

**Out of scope for the core:**
- Life-cycle optimisation across heterogeneous cohorts. The cohort protection requirement is an
  **exogenous input**.
- Inflation protection. Wtp BR is nominal by law.
- The return portfolio, beyond a premium on capital freed by swaps.

## 5. Staged plan (build in this order)

1. **Stage 1 — static base case (must work; this is the guaranteed thesis).** One period, one cohort,
   parallel rates plus one spread factor, a convex QP, and an analytic frontier. See §6.
2. **Stage 2 — richer static model.**
   - Key-rate exposures (vector hedge ratio).
   - Scenario-based CVaR, as an LP (Rockafellar–Uryasev).
   - A collateral-exhaustion chance constraint.
   - Transaction costs.
   - Capital freed by swaps earning a return-portfolio premium.
3. **Stage 3 — multi-period.** Monthly rebalancing with bands, partial adjustment toward an aim
   portfolio (Gârleanu–Pedersen 2013), and time-consistency (Basak–Chabakauri 2010). Stochastic
   programming or ADP.
4. **Stage 4 — optional.** RL or deep-hedging policy (Murray et al. 2022 featurisation), **only if
   stages 1–3 land**.

**Nesting rule.** Every extension must collapse exactly to the previous stage when its new feature
is switched off. Write that as a test.

## 6. Base-case specification (what to code first)

### 6.1 Notation
All quantities are **per unit of protected capital** $K = 1$, where $K = \sum_c \pi_c V_c$ is the
capital whose return must equal BR. Relative to $K$ the target hedge ratio is 100% by construction.
The familiar 50–80% hedge ratios are *fund-level* numbers relative to total liabilities.

| Symbol | Meaning | Unit |
|---|---|---|
| $\Delta$ | period length, $1/12$ (monthly) | years |
| $w_c$ | cash / MMF holding (market value) | fraction of $K$ |
| $w_b$ | bond holding (market value), e.g. DSL/Bund basket | fraction of $K$ |
| $n_s$ | receiver-swap notional (unfunded) | multiple of $K$ |
| $r^f$ | floating rate paid on the swap (6m Euribor) | p.a., decimal |
| $c$ | cash drag: MMF yield $= r^f - c$ | p.a., decimal |
| $y(\tau)$ | swap / RTS rate for maturity $\tau$ | p.a., decimal |
| $s$ | asset-swap spread = bond yield − matched-maturity swap rate ($>0$: bond yields more) | p.a., decimal |
| $D_b, D_s, D_L$ | modified durations of bond, swap (per unit notional), protection liability | years |
| $\Delta y$ | parallel shift of swap curve / RTS over the period | decimal |
| $\Delta s$ | change in asset-swap spread over the period | decimal |
| $\sigma_y, \sigma_s, \rho$ | vols of $\Delta y$, $\Delta s$ and their correlation | per period |
| $G$ | duration gap $= w_b D_b + n_s D_s - D_L$ | years |
| $h$ | hedge ratio $= (w_b D_b + n_s D_s)/D_L = 1 + G/D_L$ | — |

### 6.2 Instrument returns (first order: duration, no convexity, no roll-down)

$$
R_c = (r^f - c)\,\Delta, \qquad
R_b = \big(y(T_b) + s\big)\Delta - D_b\,(\Delta y + \Delta s), \qquad
R_s = \big(y(T_s) - r^f\big)\Delta - D_s\,\Delta y .
$$

$R_s$ is P&L per unit notional; a swap uses no capital. Overlay return:
$R_P = w_c R_c + w_b R_b + n_s R_s$. Note that cash plus a receiver swap is a synthetic bond earning
the fixed rate minus the cash drag.

### 6.3 Protection return (liability side)
General definition: the cohort's projected cash flows $CF_k$ at times $\tau_k$, valued on the DNB RTS:
$L_t = \sum_k CF_k\,(1+R_{t,\tau_k})^{-\tau_k}$, and $R_L = L_{t+\Delta}/L_t - 1$ with cash flows held
fixed. **Base case:** the zero-coupon shortcut of §3.3 with $d$ = liability duration, whose first-order
form is

$$
R_L \approx y_L\,\Delta - D_L\,\Delta y, \qquad y_L = y(T_L).
$$

### 6.4 Mismatch return and its moments

$$
m = R_P - R_L = \mu\,\Delta \;-\; G\,\Delta y \;-\; w_b D_b\,\Delta s .
$$

Take $E[\Delta y] = E[\Delta s] = 0$. **History gives the risk; today's curve gives the expected
return.** Using the budget $w_c + w_b = 1$, the annualised expected mismatch splits into two parts:

$$
\mu = \underbrace{w_b\,s - w_c\,c}_{\text{spread income} - \text{cash drag}}
\;+\; \underbrace{w_b\big(y(T_b)-r^f\big) + n_s\big(y(T_s)-r^f\big) - \big(y_L - r^f\big)}_{\kappa\,:\ \text{curve-carry mismatch (0 on a flat curve)}} .
$$

The variance uses factor exposures, not a raw instrument correlation matrix:

$$
\operatorname{Var}(m) = \beta^\top \Sigma_f\, \beta, \qquad
\beta = -\big(B^\top x - b_L\big), \quad
x = \begin{pmatrix} w_c \\ w_b \\ n_s \end{pmatrix}, \quad
B = \begin{pmatrix} 0 & 0 \\ D_b & D_b \\ D_s & 0 \end{pmatrix}, \quad
b_L = \begin{pmatrix} D_L \\ 0 \end{pmatrix}, \quad
\Sigma_f = \begin{pmatrix} \sigma_y^2 & \rho\sigma_y\sigma_s \\ \rho\sigma_y\sigma_s & \sigma_s^2 \end{pmatrix}.
$$

Rows of $B$ are instruments and columns are risk factors ($\Delta y$, $\Delta s$). This form extends
to key rates by adding columns. The protection return sits inside the covariance through $b_L$.
Report annualised values: $\mu_{\text{ann}} = 12\,\mu_{\text{month}}$ and
$\sigma_{\text{ann}} = \sqrt{12}\,\sigma_{\text{month}}$. The $\sqrt{12}$ scaling assumes i.i.d.
changes. Spreads mean-revert (pull-to-par), so long-horizon spread risk is overstated; that is a
stage-3 insight, not a bug.

### 6.5 Optimisation problem
For a target $\bar\mu$, solve

$$
\min_{x}\; \operatorname{Var}(m) \quad \text{s.t.}
$$

1. $\mu(x) \ge \bar\mu$ (return target; sweep $\bar\mu$ to trace the frontier).
2. $w_c + w_b = 1$ (budget on funded instruments; relaxed to $\le 1$ in stage 2).
3. $h_{lo} D_L \le w_b D_b + n_s D_s \le h_{hi} D_L$ (hedge ratio as a **band**, not an equality).
4. $w_c \ge n_s D_s\,\Delta y^{\text{str}} + \text{im}\cdot n_s$ (cash covers variation margin on the
   receivers under a stress rise $\Delta y^{\text{str}}$, plus initial margin). Base-case assumption:
   swaps are CCP-cleared and VM is paid in cash.
5. $w_b D_b \le SD_{\max}$ (**spread limit**, expressed as spread duration; confirm Cardano's form).
6. $w_c \ge 0,\; w_b \ge 0,\; n_s \ge 0$.

This is a convex QP. Solve with `cvxpy`. The equivalent penalty form is
$\max_x \mu - \tfrac{\lambda}{2}\operatorname{Var}(m)$ with a sweep over $\lambda$.

### 6.6 Closed-form results (implement as tests)
Assume a flat curve ($r^f = y$), constraint 2 binding, and the collateral constraint off unless
stated.

- **Frictionless collapse.** If $c = s = \sigma_s = 0$, then $\mu \equiv 0$ and $\sigma = |G|\sigma_y$.
  The optimum is $G = 0$, i.e. perfect replication with $m \equiv 0$ to first order.
- **Exact hedge ($G=0$).** $\mu = w_b(s + c) - c$ and $\sigma = w_b D_b \sigma_s$. The frontier is a
  **straight line**, $\mu = -c + \dfrac{s + c}{D_b\,\sigma_s}\,\sigma$. The single degree of freedom
  is the bond share $w_b$.
- **Endpoints reproduce Ernsting et al.** At $w_b = 0$, $\mu = -c$ (−25 bp/yr, −5% over 20y). At
  $w_b = 1$, $\mu = s$ (+40 bp/yr, +8% over 20y).
- **Minimum-variance duration gap** for given $w_b$: $G^* = -\,w_b D_b\,\rho\,\sigma_s/\sigma_y$. Then
  $\sigma_{\min} = w_b D_b \sigma_s \sqrt{1-\rho^2}$ and $h^* = 1 + G^*/D_L$. A non-zero rate–spread
  correlation implies a deliberate over- or under-hedge.
- **Collateral cap** (exact hedge, im = 0):
  $w_b \le \dfrac{1 - D_L\,\Delta y^{\text{str}}}{1 - D_b\,\Delta y^{\text{str}}}$. With
  $D_L = 20$, $D_b = 15$ and $\Delta y^{\text{str}} = 1\%$, this gives $w_b \le 0.941$.
- **Liability module.** Reproduce Potter van Loon & Alserda Table 1 (10.09 → 10.35; 60% × 2.17% =
  1.30%). Also reproduce the Table 2 benefit adjustments via $(1+\text{BR}_{\text{approx}})/(1+\text{BR}_{\text{tariff}})-1$:
  2022 gives −1.57% and 2023 gives +0.62%.
- **Swap sizing.** $n_s = (h D_L - w_b D_b)/D_s$.

### 6.7 Placeholder parameters (illustrative only, never report as estimates)
Use these to run the pipeline before data arrive. Every parameter object carries a
`source ∈ {"placeholder", "calibrated"}` field.

| Parameter | Value | Origin |
|---|---|---|
| $D_L$ | 20 | handoff worked example |
| $D_b$ | 15 | Ernsting et al. example |
| $D_s$ | 20 (≈30y receiver) | stylised |
| $s$ | 40 bp | Ernsting et al. example |
| $c$ | 25 bp (20 bp alt.) | Ernsting et al.; Roel |
| $\sigma_y, \sigma_s, \rho$ (annual) | 0.8%, 0.3%, 0 | arbitrary test values |
| $\Delta y^{\text{str}}$ | +100 bp | replace with Cardano's collateral stress |
| im | 0 | replace with Cardano's IM model |
| $[h_{lo}, h_{hi}]$ | [0.95, 1.05] | placeholder |
| $SD_{\max}$ | 5 | placeholder |

## 7. Data (monthly, end-of-month)

| Series | Used for | Source | Notes |
|---|---|---|---|
| DNB RTS (full curve) | $R_L$, liability durations | DNB website | Methodology changes over time (LLP, UFR / extrapolation). Store the version per date. |
| EUR swap curve vs 6m Euribor, 1–50y | $y(\tau)$, $\Delta y$, swap revaluation | Bloomberg | Candidate tickers `EUSA<n> Curncy` (verify). |
| €STR OIS curve | discounting, robustness | Bloomberg | €STR exists only from Oct 2019. |
| 6m Euribor, €STR fixings | $r^f$, cash return | Bloomberg | e.g. `EUR006M Index`, `ESTRON Index` (verify). |
| DSL / Bund yields or ASW spreads by maturity | $s$, $\Delta s$, bond revaluation | Bloomberg | e.g. `GTNLG<n>Y Govt`, `GTDEM<n>Y Govt` (verify). |
| MMF yield vs €STR / Euribor | cash drag $c$ | Bloomberg / Cardano | |
| Cohort cash flows | $D_L$, later key-rate profile | Cardano, or stylised with the AG Prognosetafel | Confidential if from Cardano. |
| Collateral stress move, IM | constraint 4 | Cardano collateral management | |
| Swaption implied vols | stage 2+ only | Bloomberg | |

**Estimation.**
- $\sigma_y$, $\sigma_s$, $\rho$ come from monthly changes: full sample, rolling 5y, and EWMA as
  robustness.
- Expected returns come from today's curve and spreads.
- **Monte Carlo** with a calibrated rate model (Hull-White / AFNS) is needed only once swaptions,
  CVaR, exhaustion probabilities or multi-period rebalancing enter. Scenarios need real-world ($P$)
  dynamics; pricing options on those paths needs risk-neutral ($Q$) calibration.

## 8. Validation and attribution (build in from day one)

- **Historical backtest by full revaluation.** Price the actual bonds and swaps off the curves each
  month, not with duration approximations, over a sample that **includes 2022**. Compare realised
  mismatch with the model's $\mu$ and $\sigma$.
- **Ex-post mismatch attribution**, aligned with the deviation sources DNB and Cardano name:
  $$
  m_t = \underbrace{(w_b s - w_c c)\Delta}_{\text{spread income, cash drag}}
  + \underbrace{\kappa\,\Delta}_{\text{curve carry}} \;
  \underbrace{-\,G\,\Delta y_t}_{\text{duration gap}} \;
  \underbrace{-\,w_b D_b\,\Delta s_t}_{\text{spread}}
  + \underbrace{\varepsilon^{\text{curve}}_t}_{\text{non-parallel}}
  + \underbrace{\varepsilon^{\text{conv}}_t}_{\text{convexity, residual}} .
  $$
  Here $\kappa$ is the curve-carry mismatch term of §6.4. The components must sum to the total;
  test that the residual is small.
- **Out-of-model evaluation, not only out-of-sample.** Frontiers fitted on calm periods get tested on
  2022; later, policies trained under one rate model get tested under another.

## 9. Code conventions

**Repo layout** (proposed; `Data/`, `Model/` and `Docs/` already exist):
```
Fulcrum/
├── Data/
│   ├── raw/            # untouched Bloomberg / DNB exports — NEVER committed
│   ├── processed/      # cleaned monthly panels (parquet) — not committed
│   └── README.md       # per file: source, ticker/field, pull date
├── Docs/CLAUDE.md
├── Model/              # importable package: logic lives here, not in notebooks
│   ├── params.py       # dataclasses: MarketState, Instruments, Constraints (+ source field)
│   ├── instruments.py  # cash, bond, receiver swap: carry, durations, rows of B
│   ├── liability.py    # BR: zero-coupon shortcut, tariff-based, cash-flow PV on RTS
│   ├── factors.py      # Δy, Δs panels; Σ_f estimation (sample, rolling, EWMA)
│   ├── optimise.py     # μ(x), Var(x), QP frontier (cvxpy)
│   ├── backtest.py     # monthly full revaluation, realised mismatch
│   └── attribution.py  # carry/drag, duration gap, curve, spread, residual
├── notebooks/          # exploration and figures only
├── tests/              # pytest: every item in §6.6 plus nesting tests
├── requirements.txt
└── .gitignore
```

**Environment.** `.venv`, Python 3.13. It currently has numpy, pandas and ipykernel. The base case
also needs `scipy cvxpy matplotlib pytest openpyxl pyarrow`. Pin versions in `requirements.txt`.

**Units and signs.**
- Store all rates, spreads and returns as **decimals p.a.** internally (40 bp = `0.0040`). Convert to
  bp only in reporting and plots.
- Durations are in years. Shocks are decimals (+100 bp = `0.01`).
- Rate exposure is $-\frac{1}{V}\frac{\partial V}{\partial y}$, which is **positive** for long bonds,
  receiver swaps and the liability.
- $m > 0$ means the overlay beat the protection return.
- $s$ = bond yield − swap rate (note that the US "swap spread" has the opposite sign).

**Style.**
- Variable names mirror the symbols in §6 (`w_b`, `n_s`, `D_L`, `sigma_s`, `rho`).
- Every function docstring states the formula it implements and its units.
- Pure functions for the maths; I/O kept separate.
- Seeds fixed for any simulation.
- Code and comments in English. Dutch terms only as defined in §12.

**Data hygiene.**
- Bloomberg data is licensed and must not be redistributed.
- Cardano client data is confidential.
- Neither goes into git. Add `Data/raw/` and `Data/processed/` to `.gitignore`, and check the GitHub
  repo's visibility.

## 10. Open questions for Cardano (they change the model, so ask early)

1. Does Cardano measure the gap between theoretical and realised BR? How large is it in bp? That
   number is the motivation in one line.
2. Is the required rate-exposure profile handed down (exogenous), or part of the advice?
3. Which BR method do the client funds use (indirect is most common), and what are the exact
   tariff / annuity-factor conventions?
4. Which DNB RTS variant is used for BR? How does it behave beyond the last liquid point (an
   unhedgeable mismatch component)? Is any Euribor → €STR transition planned?
5. How is the **spread limit** expressed: spread duration, % of hedge, or a TE budget?
6. Collateral: which stress scenario sizes the liquidity buffer? Is VM cash-only (CCP) or are bonds
   eligible? What IM model and repo haircuts apply?
7. Hedge-ratio band and rebalancing trigger policy.
8. How is cohort-level protection aggregated into the fund-level hedge?
9. Is there a realistic cohort cash-flow / key-rate profile for a representative client?
10. Is the linear vs convex (swaption) split modelled, or client preference?

## 11. Literature (full annotated list in the claude.ai project: `literature-shortlist-LDI-overlay-wtp.md`)

**Closest and practitioner:**
- Ernsting, Teunissen & Vereijken (2025), "Is er nog ruimte voor obligaties binnen de
  beschermingsportefeuille?", *VBA Journaal* 160, 39–43.
- Potter van Loon & Alserda (2025), "Beschermingsrendement: Theoretisch of praktisch?",
  *VBA Journaal* 160, 30–36.

**Positioning:**
- Sharpe & Tint (1990), "Liabilities—A New Approach", *JPM* 16.
- Hoevenaars, Molenaar, Schotman & Steenkamp (2008), *JEDC* 32.
- Ang, Chen & Sundaresan, "LDI with downside risk".
- Redington (1952).

**Wtp:**
- Werker (2022), Netspar Board Brief.
- DNB Open Boek Toezicht on managing interest rate protection in the SPR.
- van Gastel, Kortleve, Nijman & Schotman (2024).
- Bovenberg & Mehlkopf (2014).

**Frictions:**
- Jansen, Klingler, Ranaldo & Duijm (2026), "Pension Liquidity Risk", *RFS*.
- Brunnermeier & Pedersen (2009).
- Gârleanu & Pedersen (2011).
- Klingler & Sundaresan (2019).
- Pinter, Siriwardane & Walker (2025).
- BoE (2023) on the 2022 gilt crisis.

**Method:**
- Ho (1992), key rate durations.
- Rockafellar & Uryasev (2000), CVaR.
- Gârleanu & Pedersen (2013).
- Basak & Chabakauri (2010).
- Murray et al. (2022) and Buehler et al. (2019), stage 4 only.

**Background only:** Ahmadi & Godin (2025), swaption hedging (see §13).

Local PDFs: `Cardano/Thesis/Papers/Overlay portfolio management optimization`.

## 12. Glossary (NL → EN, as used in code and text)

| Dutch | English / meaning |
|---|---|
| Wet toekomst pensioenen (Wtp) | Future Pensions Act |
| Solidaire / flexibele premieregeling (SPR / FPR) | solidarity / flexible contribution scheme |
| Beschermingsrendement (BR) | protection return |
| Overrendement (OR) | excess return |
| Mismatchrendement | mismatch return = realised − intended BR |
| Projectierendement | projection return |
| Rentetermijnstructuur (RTS) | DNB term structure of interest rates |
| Theoretische (indirecte) / praktische (directe) methode | theoretical (indirect) / practical (direct) BR method |
| Annuïteitsfactor / tarief | annuity factor (capital per €1 of benefit) |
| Beschermingsportefeuille / rendementsportefeuille | protection (matching) / return portfolio |
| Renteafdekking, afdekkingspercentage | interest rate hedge, hedge ratio |
| Toedelingsregels | allocation rules |
| Leenrestrictie | borrowing restriction |
| Dekkingsgraad | funding ratio (FTK) |
| Zekerheidsequivalent | certainty equivalent |

## 13. Lessons already learned (do not repeat)

- **A pension fund's receiver swaption is a hedging instrument, not a liability to hedge.**
  Delta-hedging it removes the protection it was bought for. The dealer framing of Ahmadi & Godin
  (2025) does not transfer to buy-side LDI.
- Payer vs receiver is a sign flip (swaption put–call parity) and **not** a contribution.
- Do not start from a method (RL) and look for a problem.
- Mean–variance objectives are time-inconsistent over multiple periods. Use CVaR or exponential
  utility when going multi-period.
- Call myopic hedging "myopic", not "static".

## 14. How Claude should work on this project

- **Problem first.** Tie every modelling choice to the frictions in §3.4–3.5.
- **Build up from the general formula.** State the general definition, then derive the specific
  case. Define **every symbol** (meaning and unit) the first time it appears. Keep explanations brief
  but clear.
- **Stage 1 before anything else.** No extension until the base case passes all §6.6 tests. Each
  extension ships with its nesting test.
- **Closed forms first.** Where one exists, test numerics against it.
- **Never invent market data or estimates.** If a series is missing, say so and use a labelled
  placeholder.
- Mathematics in LaTeX. Figures reproducible from code in `Model/`.
- Do not commit data. Do not push without asking.

## 15. Status (29 Sep 2026)

- Repo initialised; no model code yet.
- Base-case formulation agreed in discussion (Sept 2026) and recorded in §6.
- Thesis proposal in progress; working title (provisional): *Risk–return optimisation of LDI overlay
  strategies under collateral and liquidity constraints*.
- **Next step:** implement `instruments.py`, `liability.py` and `optimise.py` with placeholder
  parameters and the §6.6 tests. Then pull the §7 data and calibrate.

---

## 16. Weekly log (auto-updated every Friday)

Appended automatically from Frank's Claude app chats; newest entry last. Where an entry conflicts with anything above (including §15 Status), the entry takes precedence.

<!-- weekly-log:start -->

### Week ending 2026-10-02

_Source: Claude memory only (chat-history search tools were not available in this run); dates of some items are uncertain._

- **Decisions**
  - Changes §5 (Stage 3) and thesis structure: thesis agreed as **two chapters** — (1) the one-period mean–tracking-error base case (Markowitz, §6); (2) a dynamic extension as a **Merton-type multi-period stochastic optimisation**. Prompted by Roel's suggestion to add the mismatch as a second, *rewarded* Brownian motion next to equity risk, solved numerically (cf. his paper "Optimal design of pension fund schemes"). Frank has not used Merton before.
  - Adds to §4 risk axis / §6.5 (under consideration, not decided): replace the variance of $m$ by a **downside risk measure**, since falling short of BR harms the fund while exceeding it benefits the OR.
- **Progress**
  - Writing: Overleaf base-case draft (date uncertain, possibly before this window) — hypothetical cohort of 100 participants aged 40, contributing from 25 to 68, benefits until 100; objective $\mathbb{E}[R_{\text{mismatch}}(x)] - \lambda\,\operatorname{Var}[R_{\text{mismatch}}(x)]$ with $R_{\text{mismatch}} = R_{\text{overlay}} - R^L$, under the §6.5 constraints.
  - Overleaf notation (differs from §6.1; reconcile before coding): $V^L_t$ liability value, $V^O_t$ overlay value, $CF_{t\to t+1}$ cash flows, $A_t$ protected capital ($= K$ in §6.1). Liability = one survival-weighted cash-flow vector of benefits on accrued entitlements only (zero before 68); contributions change $A_t$ and the cash flows *between* periods, not the within-period return. With a fully funded overlay $V^O_t = A_t$ at the start of each period.
  - Supervisors: adds to §2 — **Johan van Leeuwaarden** is the university supervisor (stochastic optimisation / OR).
  - Code: no model code reported yet beyond §15.
- **Cardano input**
  - No answers to §10 questions found this week.
- **Next steps**
  - Write an initial mathematical model (both chapters) on paper and discuss with Roel and Johan.
  - Then implement `instruments.py`, `liability.py`, `optimise.py` with the §6.6 tests (unchanged from §15).

### Week ending 2026-10-09

_Source: Claude memory only (chat-history search tools were not available in this run); only one thesis memory file changed in this window, so other work this week may be missing._

- **Decisions**
  - Adds to §4 / §6.5 and the chapter-2 (Merton) extension — leaning towards, not decided: treat the protection return (BR) as the **risk-free rate / numéraire** of the model, motivated by Sharpe & Tint (1990).
- **Progress**
  - Literature (adds to §11): read **Sharpe & Tint (1990), "Liabilities — A New Approach", JPM 16(2)**. Key points from Frank's notes: asset–liability comovement is a utility benefit (i.e. a hedge); the *liability hedging credit* (LHC) measures how much an asset provides utility to an investor with given liabilities; surplus optimisation accepts lower expected return and/or more asset risk to better hedge liability increases; drop terms in the objective that are constant w.r.t. the decision. Verdict: very elegant; the risk / return / hedging-credit decomposition is a candidate framing for the mismatch objective.
  - Thesis quotes saved in Frank's reading notes (Sharpe & Tint on surplus optimisation and on the LHC as the one extra component).
  - Code / writing / meetings: nothing new found in memory this week.
- **Cardano input**
  - No answers to §10 questions found this week.
- **Next steps**
  - Open questions from the reading: can the risk / return / LHC formulation be carried into the Merton model? Why does Sharpe–Tint use mean–variance utility and Merton CRRA, and which utility functions are candidates (links to the downside-risk question from last week)?
  - Continue the literature review (Merton 1969 is listed in the reading notes but has no notes yet).
  - Initial mathematical model for both chapters to discuss with Roel and Johan; then `instruments.py`, `liability.py`, `optimise.py` with the §6.6 tests (unchanged).

### Week ending 2026-10-16

_Source: Claude memory only (chat-history search tools were not available in this run); the two thesis memory files updated this week carry no per-item dates, so attribution to this week is likely but not certain._

- **Decisions**
  - Changes §5 / §15 (sequencing): Frank submitted an initial LaTeX report (Merton model extended with mismatch risk) to Roel and Johan; **both agreed to build an initial "Merton" model first**.
  - Supersedes §15 "Next step" — plan: (1) code the base case with $\theta_E = 0$, $\beta = \omega = 1$ (symbols as defined in Frank's LaTeX report, not yet in §6; reconcile) under Ernsting et al.'s **policy 1 (no spread risk)** and **policy 2 (zero carry)** (§3.4), over ~two weeks, to show the supervisors; (2) derive the HJB equation and the optimal bond-share policy $x_B$; (3) the full model.
- **Progress**
  - Writing: initial LaTeX report on the Merton-with-mismatch model sent to both supervisors (see above).
  - Literature (adds to §11): read **Merton (1969)**, "Lifetime Portfolio Selection under Uncertainty: The Continuous-Time Case", *REStat* 51(3). Frank's notes: Merton is the continuous-time version of the mean–variance tangency logic; CRRA or CARA utility is needed to solve the Bellman system analytically; infinite-horizon conditions are easier, so examine the infinite-horizon problem first; admissible processes must be Markovian with first two moments $O(\Delta t)$ and higher moments $o(\Delta t)$.
  - Working rule (adds to §14): LLM use is allowed but output must not be adopted blindly; Frank must be able to derive and defend every modelling choice to his supervisors (duration was not part of his curriculum, so explain it from first principles).
  - Code: no model code reported yet; base-case coding (VS Code + Claude Code) is the next step.
- **Cardano input**
  - No answers to §10 questions found this week.
- **Next steps**
  - Implement the base case ($\theta_E = 0$, $\beta = \omega = 1$; policies 1 and 2) for the supervisors in ~two weeks; then HJB / optimal $x_B$.
  - Open questions from the reading: when do KKT conditions apply (and are they only analytical)? What do "Markovian" and "class of probability distributions" mean in Merton (1969)? Why mean–variance (Sharpe–Tint) vs CRRA (Merton), and which utilities are candidates?
<!-- weekly-log:end -->
