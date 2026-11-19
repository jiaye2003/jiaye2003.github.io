---
title: "Semi-parametric Asset Pricing Model for China A-Share Market"
excerpt: "regression-based PCA methods for constructing latent return factors from managed portfolios in equity markets."
collection: portfolio
---
## Summary

This project studies regression-based PCA methods for constructing latent return factors from managed portfolios in equity markets. The pipeline includes cross-sectional feature preprocessing, managed-portfolio construction, factor extraction, and portfolio evaluation.

# 1 Methodology

This project uses a regression-based PCA method to predict future stock price. The model it used is a **conditional factor model**. The base model is as following:

$$
y_{it}=\alpha(z_{it})+\beta(z_{it})'f_t+\varepsilon_{it}.
$$

Where

- $y_{it}$ is the excess return of stock $i$ at period $t$;
- $z_{it}$ is the charactistic(feature) vector of stock $i$ at period $t$;
- $f_t$ is the vector of latent factors;
- $\beta(z_{it})$ is the factor exposure, it is assumed to be a function of $z_{it}$;
- $\alpha(z_{it})$ is the pricing error left over by this model, assumed to be a function of $z_{it}$.

The main contribution of this method is to disentangles the role of features into risk exposure (beta) and pricing error (alpha), and use them to predict future price.

The framework draws on three strands of literature.

**1. APT and Fama-French Factor Models**

The starting point is that a small number of common factors explain asset returns:

$$
r_{i,t+1} = \beta_{it}^{\top} f_{t+1} + \varepsilon_{i,t+1},
$$

**2. Conditional Factor Models**

Factor exposures can vary with asset characteristics rather than remaining constant:

$$
\beta_{it} = \beta(z_{it}),
$$

where $z_{it}$ contains characteristics observed at time $t$. Allowing for characteristic-dependent pricing errors gives:

$$
r_{i,t+1} = \alpha(z_{it}) + \beta(z_{it})^{\top} f_{t+1} + \varepsilon_{i,t+1}.
$$

**3. Semiparametric Estimation**

The functional forms of $\alpha(\cdot)$ and $\beta(\cdot)$ are generally unknown. Sieve methods approximate these functions using a finite collection of basis functions, allowing a flexible relationship between characteristics, pricing errors, and factor exposures.

## Linear Specification

In my implementation, I assume that both alpha and beta are linear in the characteristics:

$$
\alpha(z_{it}) = z_{it}^{\top} a,
\qquad
\beta(z_{it}) = B^{\top} z_{it}.
$$

The model therefore becomes:

$$
r_{i,t+1} = z_{it}^{\top} a + z_{it}^{\top} B f_{t+1} + \varepsilon_{i,t+1}.
$$

Here, $a$ captures the characteristic-related pricing-error component, while $B$ maps characteristics into factor exposures. A constant can be included in $z_{it}$ to allow for intercepts.

# 2 Data and Sample

## 2.1 Sources, universe, and sample construction

The data are from CSMAR. Raw data include stock prices and returns, trading status, market capitalization, financial statements, disclosure dates, ownership, and risk-free returns. The universe includes Shanghai and Shenzhen A-shares on the main boards, ChiNext, and the STAR Market (`markettype` 1, 4, 16, and 32); B-shares and Beijing Stock Exchange securities are excluded. The panel is unbalanced: stocks need not appear in every period. Stock-period keys are unique.

The project constructs eight panels by combining two return definitions (close-to-close and open-to-close), two financial report information timing rules (actual announcement dates, `annodt`, and disclosure deadlines, `deadline`), and two frequencies (weekly and monthly). Model estimation starts in 2006. The statistics below refer specifically to the **weekly, open-to-close, actual-announcement-date, full-market sample**, with the number of RPCA return factors fixed at **K = 10**. Weekly labels follow the ISO year-week calendar. 

The panel contains **3,179,968 stock-week records** and **5,493 stocks**, including the unfinished 2026-W23 period.

| Sample statistic        | Full-sample estimation / IS |     OOS evaluation |
| ----------------------- | --------------------------: | -----------------: |
| Evaluation period       |          2006-W01–2026-W22 | 2007-W44–2026-W22 |
| Stock-week observations |                   3,060,108 |          2,954,646 |
| Distinct stocks         |                       5,485 |              5,467 |
| Usable weeks            |                       1,037 |                947 |
| Mean stocks per week    |                     2,950.9 |            3,120.0 |
| Median stocks per week  |                       2,476 |              2,709 |
| Minimum stocks per week |                         915 |              1,255 |
| Maximum stocks per week |                       5,180 |              5,180 |

IS and OOS use separately selected feature lists, so the OOS panel is defined by its own finite-feature requirements rather than simply assumed to be a date slice of the IS panel. Calendar periods that fail the sample requirements are omitted; the training window counts usable periods.

**ST, *ST, and suspension records are not categorically removed by the current treatment.** The actual IS sample includes 52,487 stock-week observations with a nonzero suspension indicator; the OOS sample includes 40,493. Opening-price eligibility still determines whether a position can be initiated.

## 2.2 Descriptive statistics

The table reports pooled stock-week statistics for the actual model samples. Each stock-week receives equal weight; these are individual-stock return distributions, not portfolio returns or annualized moments. Returns are expressed in percent, and standard deviations use `ddof=1`.

| Sample and variable      | Mean (%) | Std. dev. (%) | 5th percentile (%) | Median (%) | 95th percentile (%) |
| ------------------------ | -------: | ------------: | -----------------: | ---------: | ------------------: |
| IS: stock return         |   0.3430 |        6.6556 |           −9.3955 |     0.0000 |             11.1225 |
| IS: stock excess return  |   0.3076 |        6.6556 |           −9.4323 |   −0.0182 |             11.0853 |
| OOS: stock return        |   0.2875 |        6.5869 |           −9.3496 |     0.0000 |             10.8936 |
| OOS: stock excess return |   0.2528 |        6.5870 |           −9.3855 |   −0.0286 |             10.8573 |

Stock excess return is the realized stock return less the same week's risk-free return. For the same 3,179,968 source records from 2006 onward, observed-cell coverage across the 37 original characteristics rises from **93.65% before imputation to 98.68% after imputation**. This coverage denominator includes the supplied panel's unfinished final period and precedes opening-price and model-sample restrictions; it is not the denominator of the return-distribution table.

## 2.3 Return definitions and information timing

For open-to-close returns, the holding interval runs from the first market trading day's open to the last market trading day's close within the week or month. The adjusted opening price is the raw opening price multiplied by the same day's adjusted-close/raw-close ratio. In the current backtest, a valid positive opening price is required. If the exact terminal quote is missing, the position is marked at the latest available adjusted closing price no later than the period end. The resulting return is terminal marked value divided by adjusted opening price, minus one. This backward-looking valuation rule does not imply that a trade could be executed at a stale quote.

Close-to-close returns use the prior available stock-period closing observation as the denominator; when a stock has missing periods, this interval can span more than one market period. The two return definitions therefore remain separate throughout estimation and reporting.

For the focal open-to-close sample, market characteristics are formed using information through the **last market trading day of the preceding period**. Financial statements are selected as of that same cutoff. In the `annodt` version, the actual announcement date determines availability; for close-to-close `annodt`, availability begins on the next calendar day. The `deadline` version instead waits until the later of the actual announcement date and the prescribed disclosure deadline, subject to report-period and historical-input availability checks. It does not make delayed announcements available early.

The most recent report period available by the cutoff is used. Historical reports needed for year-on-year and trailing-twelve-month calculations must also have known disclosure dates consistent with that cutoff. SOE status uses the latest available prior-year ownership record, with its annual-report disclosure date as an availability proxy. Close-to-close market characteristics use the preceding market period's penultimate trading day, while its financial cutoff follows the stock-specific prior return-denominator date; these dates need not coincide.

The final panel already aligns formation-period characteristics with the realized holding-period return. **No additional return lead or feature lag is applied in the RPCA backtest.** `ret_lag` is constructed by shifting the relevant return by one row within each stock's chronologically sorted history. A missing previous return remains missing; if a stock skips a market period, the lag refers to its preceding recorded observation, not necessarily the immediately preceding calendar week or month.

## 2.4 Characteristic construction and accounting treatment

There are **37 original characteristics**, plus the separately constructed `ret_lag`, giving **38 candidate predictors**. The 37 characteristics consist of:

- **25 accounting and valuation characteristics:** `AT`, `A2ME`, `ATO`, `BEME`, `C`, `CTO`, `D2A`, `DPI2A`, `E2P`, `FC2Y`, `FreeCF`, `Investment`, `Lev`, `NOA`, `OA`, `OL`, `PCM`, `PM`, `Prof`, `Q`, `RNA`, `ROA`, `ROE`, `S2P`, and `SGA2S`.
- **11 market characteristics:** `LME`, `LTurnover`, `r2_1`, `r12_2`, `r12_7`, `r36_13`, `Rel_to_high`, `Beta`, `SUV`, `Idio_vol`, and `Spread`.
- **Ownership:** `SOE`.

Financial-statement preprocessing retains consolidated statements and removes parent-company-only statements. Balance-sheet items remain point-in-time balances. Income and cash-flow items use trailing-twelve-month values: an annual report keeps its annual value; an interim report uses current year-to-date value plus the previous annual value minus the previous year's corresponding interim value. Missing required historical reports or historical disclosure dates leave the affected TTM values missing; they are not filled by temporal interpolation in this step. The existing accounting formulas retain their specified zero substitutions for some optional components, which is distinct from filling an otherwise missing report. Ratios using lagged assets or operating assets refer to the same report period one year earlier, not simply the previous quarter.

Raw `LME` is total market capitalization in thousands of RMB, despite its name; it is not a logarithm. Accounting valuation ratios retain the project's existing floating-market-capitalization denominator and accounting-unit conventions. Thus their denominators should not be assumed to equal raw `LME`.

Market windows are defined by dates, including for weekly observations:

| Characteristic  | Construction through the formation signal date                                                                                                                                                                                                            |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Beta`        | Correlation times the stock/market volatility ratio; volatility uses 252 market trading days with at least 120 observations, and correlation uses overlapping three-day log excess returns over 1,260 trading days with at least 750 paired observations. |
| `Idio_vol`    | Sample standard deviation of daily excess-return residuals from a local FF3 regression over the preceding natural month, requiring at least 15 complete observations.                                                                                     |
| `LTurnover`   | Shares traded over the preceding natural month divided by total shares outstanding at the signal date; the left endpoint is excluded and the signal date is included.                                                                                     |
| `Spread`      | Mean daily amount-weighted relative quoted spread over the preceding natural month, including both endpoints.                                                                                                                                             |
| `Rel_to_high` | Adjusted closing price divided by the highest adjusted daily high in the preceding 364-calendar-day window, with the left endpoint excluded.                                                                                                              |
| `SUV`         | Formation-month-to-date abnormal trading volume, using an expanding volume/positive-return/negative-return regression fitted only through the previous month, with at least 60 observations.                                                              |

For momentum, let m denote the natural month containing the signal date, and let P(k) be the adjusted closing price at month k's final market trading day, or the most recent valid price before that date. Then `r12_2 = P(m−1)/P(m−12)−1`, `r12_7 = P(m−6)/P(m−12)−1`, and `r36_13 = P(m−12)/P(m−36)−1`. `r2_1` runs from the previous month-end price to the signal-date price. These subscripts refer to months even in the weekly panel. Missing endpoint quotes are searched backward only, with no maximum lookback; missing intermediate month-end quotes do not invalidate an otherwise available endpoint return. Before-listing histories remain missing. A long suspension can cause both endpoints to use the same old quote.

The local FF3 factors used to construct idiosyncratic volatility are formed each June using a 2×3 size/book-to-market sort and held through the following June. Portfolio returns use lagged floating-market-capitalization weights; the required annual report must be disclosed by formation. These are locally constructed Chinese factors, not assumed to reproduce an external vendor's factor series exactly.

## 2.5 Cross-sectional transformation and missing characteristics

Before imputation, each original characteristic is transformed within its weekly or monthly cross-section using average ranks for ties:

$$
z_{ijt}=\frac{\operatorname{rank}_t(x_{ijt})-1}
{\max_i\operatorname{rank}_t(x_{ijt})-1}-\frac12.
$$

Observed ranks lie in **[−0.5, 0.5]**, not the half-open interval previously stated. When the maximum rank is at most one, observed entries are assigned zero; missing entries remain missing. There is no additional winsorization of the raw panel. Ranking changes characteristic scale rather than trimming extreme observations.

Missing ranked characteristics are filled separately in each time cross-section using the existing PCA characteristic-imputation procedure, with **seven imputation factors and a minimum of ten observed characteristics per stock**. Pairwise observed characteristic products estimate a second-moment matrix; its leading positive-eigenvalue directions give loadings, and each stock's available characteristics identify its factor scores. Only missing entries are replaced by fitted values. Observed ranks remain unchanged. Stocks with too little information and characteristics unobserved across the eligible cross-section can retain missing values. Model-generated fills are not clipped, so some can fall outside the original rank interval.

Stock identifiers, dates, returns, trading-status flags, market classifications, disclosure metadata, suspension flags, and risk-free returns are not ranked or imputed. `ret_lag` is added separately and retains return units. `SOE` does enter the characteristic ranking and imputation step, so that predictor is no longer a binary ownership label; ownership-based grouping instead uses the raw availability-matched label.

The all-market RPCA model uses these supplied characteristic values without a second rank transformation. The size and other block-based models re-rank the selected predictors within each period and block. The **imputation dimension of seven** is distinct from the **return-factor dimension of ten**. Seven was selected by masked-characteristic validation over the full 2006-onward calibration sample. Consequently, the reported OOS results are conditional on that previously calibrated input pipeline, not a fully nested OOS test of imputation-model selection.

## 2.6 Feature selection and estimation windows

Each fitted model uses **30 selected predictors plus an intercept**, producing 31 managed-portfolio columns in the all-market case. `LME`, `BEME`, `Beta`, and `ret_lag` are mandatory; 26 additional predictors are selected from the remaining candidates. Selection uses a common complete-case panel across the 38 candidates, at least 50 stocks per cross-section, and at least 24 valid Fama–MacBeth periods.

Forward selection ranks each incremental candidate by the absolute Newey–West/HAC t-statistic of its cross-sectional coefficient series. The automatic lag choice is the integer part of \(4(T/100)^{2/9}\). Candidates with absolute pooled Pearson correlation of at least 0.8 with an already selected predictor are removed; adjusted-R² gains are retained as diagnostics and tie-break information. The threshold is not relaxed to force a desired result.

IS feature selection and parameter estimation use the full eligible sample. For the focal weekly OOS experiment, feature selection uses only the **first 90 usable periods, 2006-W01 through 2007-W43**, and the resulting list is fixed. Evaluation starts at **2007-W44**. At each subsequent test period, RPCA loadings, alpha, factor moments, and portfolio optimizer weights are re-estimated on all preceding usable periods. K remains ten throughout; it is not selected by comparing OOS Sharpe ratios. Model-specific finite-feature and completed-return restrictions are then applied when constructing managed portfolios. Rank-deficient block regressions use the original-design Moore–Penrose minimum-norm solution, with the same operator used to map portfolio weights back to stocks.

Risk-free returns are read from the corresponding panel at the same holding-period frequency, in decimal units, without another percentage or maturity conversion. The existing trading strategies are fitted to raw stock-return managed portfolios. To match the excess-return model in Section 1, the evaluation run additionally constructs and fits a separate excess-return RPCA model using \(y_{it}=r_{it}-r_{f,t}\). The R² results below refer to that excess-return model; its fit is not silently substituted into the trading strategies.

## 2.7 Model evaluation: reconstruction versus prediction

Let \(e_{it}=y_{it}-\widehat y_{it}\), let \(\mathcal T_i\) be stock i's observed evaluation periods, and let \(\mathcal I_t\) be the observed stocks in period t. The three reported R² measures use **zero return as the denominator benchmark**, without subtracting a sample mean:

$$
R^2_{\mathrm{total}}
=1-\frac{\sum_t\sum_{i\in\mathcal I_t}e_{it}^2}
{\sum_t\sum_{i\in\mathcal I_t}y_{it}^2},
$$

$$
R^2_{\mathrm{TS}}
=1-\frac{1}{N}\sum_i
\frac{\sum_{t\in\mathcal T_i}e_{it}^2}
{\sum_{t\in\mathcal T_i}y_{it}^2},
\qquad
R^2_{\mathrm{CS}}
=1-\frac{1}{T}\sum_t
\frac{\sum_{i\in\mathcal I_t}e_{it}^2}
{\sum_{i\in\mathcal I_t}y_{it}^2}.
$$

Time-series R² weights stocks equally, whereas cross-sectional R² weights evaluation periods equally. Groups with a zero denominator are excluded from the corresponding mean, and valid-group counts are recorded. For M evaluated stock-period observations,

$$
\operatorname{RMSE}=\sqrt{\frac{1}{M}\sum_t\sum_{i\in\mathcal I_t}e_{it}^2}.
$$

For **reconstruction**, the fitted return uses realized-period factors:

$$
\widehat y^{\mathrm{rec}}_{it}
=z_{i,t-1}^{\top}\widehat a_{t-1}
+z_{i,t-1}^{\top}\widehat B_{t-1}\widehat f_t.
$$

In OOS reconstruction, alpha and loadings come only from historical data, but the realized factor \(\widehat f_t\) is extracted from period-t realized returns. It measures explanatory fit with historically estimated loadings, **not an ex ante return forecast**.

For **prediction**, the realized factor is replaced by its mean in the preceding training window:

$$
\widehat y^{\mathrm{pred}}_{it}
=z_{i,t-1}^{\top}
\left(\widehat a_{t-1}+\widehat B_{t-1}\overline f_{t-1}\right).
$$

No test-period return enters this prediction. In this unconstrained linear specification, the coefficient in parentheses equals the training mean of managed returns. The IS counterpart uses full-sample moments and is reported separately from the expanding-window OOS forecast.

For the focal weekly open-to-close, `annodt`, full-market sample:

| Evaluation                 | Total R² (%) | Time-series R² (%) | Cross-sectional R² (%) | RMSE (percentage points) |
| -------------------------- | ------------: | ------------------: | ----------------------: | -----------------------: |
| IS reconstruction          |       37.2161 |             29.9304 |                 28.4668 |                   5.2793 |
| OOS reconstruction         |       36.3193 |             34.4631 |                 27.6445 |                   5.2603 |
| IS mean-factor prediction  |        0.2613 |              0.1233 |                  0.7645 |                   6.6540 |
| OOS mean-factor prediction |      −0.3022 |            −0.2991 |                  0.1876 |                   6.6018 |

The positive OOS reconstruction R² should therefore not be interpreted as equally strong return predictability: the OOS pooled predictive R² is slightly negative relative to the zero-excess-return benchmark. This distinction also prevents a high portfolio Sharpe ratio from being conflated with accurate forecasts of individual-stock returns.

Portfolio Sharpe ratios use annualized excess returns: weekly values multiply the excess-return mean/standard-deviation ratio by \(\sqrt{52}\), with `ddof=0`. For stock weights w, excess portfolio return is \(w^{\top}r-r_f\mathbf1^{\top}w\). The raw-weight `total` strategy and separately normalized long-short legs are reported as different portfolios. Transaction costs are not deducted. Source-date checks and historical estimation reduce timing errors, but the pre-calibrated imputation dimension, unavailable historical revision vintages, and stale-price valuation prevent a claim that all forms of look-ahead or execution bias have been eliminated.

# 3 Trading Strategies

This section reports the project's **linear RPCA** strategies for **weekly open-to-close returns, actual announcement dates (`annodt`), the full-market sample (`all`), and K = 10**. IS evaluation covers **1,037 weeks, 2006-W01–2026-W22**. OOS evaluation covers **947 weeks, 2007-W44–2026-W22**, following the initial 90 usable training periods. OOS characteristics are selected once in that initial window; model parameters and optimizer moments are subsequently re-estimated using an expanding historical window, as described in Section 2.

The trading strategies use **raw-return managed portfolios**. Excess returns are used when calculating Sharpe ratios. The separately fitted excess-return model used for the R² evaluation in Section 2 does not replace the trading model here.

## 3.1 Managed portfolios and the pure-alpha strategy

Let \(r_t\) be the vector of realized raw stock returns and \(Z_{t-1}\) the formation-information design matrix, containing the 30 selected predictors and an intercept. Managed returns are

$$
g_t=Z_{t-1}^{+}r_t,
$$

where \(+\) denotes the Moore–Penrose inverse. With a full-rank design, this equals \((Z_{t-1}^{\top}Z_{t-1})^{-1}Z_{t-1}^{\top}r_t\). The same regression operator is used to map every managed-portfolio direction back into stock weights.

At test period t, PCA is estimated from demeaned historical managed returns. With orthonormal loading columns \(B_{t-1}\), define

$$
a_{t-1}=(I-B_{t-1}B_{t-1}^{\top})\overline g_{t-1},
\qquad
f_t=B_{t-1}^{\top}g_t.
$$

The pure-alpha direction is \(h_{\alpha,t-1}=a_{t-1}\), giving the unscaled payoff

$$
R^0_{\alpha,t}=a_{t-1}^{\top}g_t,
\qquad
w^0_{\alpha,t}=(Z_{t-1}^{+})^{\top}a_{t-1}.
$$

The current realized return determines the payoff, but it is not needed to determine these OOS weights. The alpha direction is orthogonal to the estimated loading space; this alone does not establish that the resulting stock portfolio is market-neutral or free of omitted risks.

## 3.2 Factor MVE strategy

Using the current training-window loadings, the entire historical factor-return series is constructed from that same training window. Let \(\mu_{f,t-1}\) and \(\Sigma_{f,t-1}\) denote its sample mean and covariance. The factor mean–variance direction is

$$
q_{f,t-1}=\Sigma_{f,t-1}^{-1}\mu_{f,t-1},
\qquad
h_{\mathrm{MVE},t-1}=B_{t-1}q_{f,t-1}.
$$

Its unscaled payoff is \(R^0_{\mathrm{MVE},t}=q_{f,t-1}^{\top}f_t\). The covariance uses `ddof=1`. This optimizer uses the raw-return moments of the implemented strategy and does not impose a unit-budget or fixed-gross-exposure constraint.

## 3.3 Combined Alpha-MVE strategy

The combined strategy jointly optimizes the alpha portfolio and all ten factors. For each historical training period s, form

$$
x_s=
\begin{bmatrix}
a_{t-1}^{\top}g_s\\
B_{t-1}^{\top}g_s
\end{bmatrix},
\qquad
q_{x,t-1}=\operatorname{Cov}_{t-1}(x_s)^{-1}
\operatorname{E}_{t-1}(x_s).
$$

The managed-portfolio direction and unscaled test-period payoff are

$$
h_{\mathrm{Alpha\text{-}MVE},t-1}
=[a_{t-1},B_{t-1}]q_{x,t-1},
\qquad
R^0_{\mathrm{Alpha\text{-}MVE},t}
=h_{\mathrm{Alpha\text{-}MVE},t-1}^{\top}g_t.
$$

This is an eleven-asset optimizer within the linear model. Each OOS training series is rebuilt using the current historical estimates of alpha and loadings; it is not assembled from older OOS payoffs estimated under different parameter windows.

## 3.4 Original-weight versus normalized long-short portfolios

For each strategy direction h, the implemented stock weights are

$$
w_t=c_t(Z_{t-1}^{+})^{\top}h_{t-1},
\qquad
c_t=\frac{1}{\widehat\sigma_{m,t-1}}.
$$

Here \(\widehat\sigma_{m,t-1}\) is the sample standard deviation (`ddof=1`) of the full-market, formation-capitalization-weighted weekly return over the actual training window. IS uses its full-sample value; OOS uses only earlier training periods. This scaling applies to final stock weights, not to the managed returns or PCA loadings. It does **not** target a common strategy volatility, impose full investment, or cap gross leverage.

The results distinguish two portfolios:

**Original-weight portfolio (`total`).** This retains the optimizer's relative exposures and the market-volatility scaling:

$$
R_t^{\mathrm{total}}=\sum_i w_{it}r_{it}.
$$

It is the stock-position profit and loss under those weights, not automatically the return of a fully invested, unlevered account. The unconstrained strategies can have substantial and time-varying leverage.

**Normalized long-short portfolio (`long_short`).** Let

$$
L_t=\sum_{w_{it}>0}w_{it},
\qquad
S_t=-\sum_{w_{it}<0}w_{it}.
$$

When both legs are present, the long and short payoffs are

$$
R_t^L=\frac{\sum_{w_{it}>0}w_{it}r_{it}}{L_t},
\qquad
R_t^S=\frac{\sum_{w_{it}<0}w_{it}r_{it}}{S_t},
\qquad
R_t^{LS}=R_t^L+R_t^S.
$$

The short payoff already contains the short-position sign. This convention assigns **one unit to the long leg and one unit to the short leg: gross exposure two and net exposure zero**. It does not divide the sum of the legs by two. All stocks with eligible positive or negative strategy weights enter the corresponding leg; these results are not top/bottom-quantile or Top-100 portfolios.

In particular,

$$
R_t^{\mathrm{total}}=L_tR_t^L+S_tR_t^S,
\qquad
R_t^{LS}=R_t^L+R_t^S.
$$

Normalizing each leg removes the common positive multiplier \(c_t\), but also changes the strategy's long and short exposure magnitudes. Because those exposures vary over time, the two reported Sharpe ratios need not coincide.

For either stock-weight convention, the Sharpe calculation uses

$$
R_t^e=w_t^{\top}r_t-r_{f,t}\mathbf1^{\top}w_t,
\qquad
SR_{\mathrm{ann}}=\sqrt{52}\,
\frac{\operatorname{mean}(R_t^e)}{\operatorname{std}(R_t^e;\,\mathrm{ddof}=0)}.
$$

The exposure adjustment is equivalent to completing a unit-net-asset account with cash or financing at the same risk-free rate and then subtracting that rate once. A normalized long leg subtracts rf, a normalized short leg adds rf, and the two adjustments cancel for the net-zero normalized long-short portfolio. Sharpe ratios are dimensionless; they are not percentage returns.

## 3.5 Actual backtest results

The following annualized Sharpe ratios are taken from the saved K = 10 run for the exact sample specified above. They summarize the implemented strategies before transaction costs.

| Strategy  | IS original-weight SR | OOS original-weight SR | IS normalized long-short SR | OOS normalized long-short SR |
| --------- | --------------------: | ---------------------: | --------------------------: | ---------------------------: |
| MVE       |                3.9684 |                 3.0816 |                      4.0093 |                       3.1492 |
| Alpha     |                4.1015 |                 3.9857 |                      3.8718 |                       3.9020 |
| Alpha-MVE |                5.7072 |                 4.7962 |                      5.6179 |                       4.7671 |

Alpha-MVE has the highest OOS Sharpe among these three strategies under both conventions: **4.7962 for original weights and 4.7671 for normalized long-short weights**. Pure alpha also exceeds factor MVE in this sample. These are empirical results of the present linear implementation; they do not require a nonlinear model.

For scale, the normalized OOS MVE, Alpha, and Alpha-MVE portfolios have mean weekly payoffs of **0.4476%, 0.2934%, and 0.4824%**, respectively, and annualized volatility of **7.3912%, 3.9106%, and 5.2619%**. These values use the gross-two convention defined above; they are not the moments of the original-weight portfolios. The volatility figures use `ddof=0` and the same square-root-of-52 annualization.

## 3.6 Stock-weight nonnegative variants

The run also evaluates two constrained variants. `MVE-nonnegative` combines the full-market long-only portfolio with ten factors; `MVE-alpha-nonnegative` additionally includes the alpha portfolio. The market anchor uses formation-date capitalization weights. For the relevant training-return mean \(\mu\), covariance \(\Sigma\), and current stock mapping M, the optimizer solves

$$
\min_q\;\frac12 q^{\top}\Sigma q-\mu^{\top}q
\quad\text{subject to}\quad Mq\geq0.
$$

The constraint is on mapped **stock weights**, not on the signs of the factor coefficients. No full-investment condition is imposed. The same historical market-volatility scaling is applied afterward. Their `long_leg` results normalize positive weights to sum to one; their `total` results retain the optimizer's scale. A normalized long-short result is not defined for these portfolios because there is no short leg.

| Strategy              | IS original-weight SR | OOS original-weight SR | IS normalized long-only SR | OOS normalized long-only SR |
| --------------------- | --------------------: | ---------------------: | -------------------------: | --------------------------: |
| MVE-nonnegative       |                0.4843 |               −0.1032 |                     0.5010 |                      0.1746 |
| MVE-alpha-nonnegative |                0.5011 |               −0.0692 |                     0.5136 |                      0.2035 |

The constrained OOS results are much weaker in this specification. Normalization can change their Sharpe because it removes time-varying total exposure; it is not simply a fixed rescaling of the entire return series. The high unconstrained long-short Sharpe should therefore not be presented as evidence of comparable long-only performance.

These results do not by themselves establish market inefficiency or an executable arbitrage. Transaction costs, borrow fees, and real-world financing spreads are not deducted, short-sale availability is not enforced for the unconstrained portfolios, and stale terminal quotes are valuation marks rather than guaranteed execution prices. The imputation-calibration and timing qualifications in Section 2 also apply. High portfolio Sharpe and the slightly negative OOS individual-stock predictive R² answer different questions and can coexist.

Source: the saved `performance_K10.csv`, filtered to `case = open_to_close/annodt/weekly`, `scenario = all`, and `K = 10`. IS and OOS results use their respective feature-selection and estimation windows; the table does not select K or the reporting sample by maximizing OOS performance.
