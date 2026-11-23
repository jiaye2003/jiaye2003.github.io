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

- $$y_{it}$$ is the excess return of stock $$i$$ at period $$t$$;
- $$z_{it}$$ is the charactistic(feature) vector of stock $$i$$ at period $$t$$;
- $$f_t$$ is the vector of latent factors;
- $$\beta(z_{it})$$ is the factor exposure, it is assumed to be a function of $$z_{it}$$;
- $$\alpha(z_{it})$$ is the pricing error left over by this model, assumed to be a function of $$z_{it}$$.

The main contribution of this method is to disentangles the role of features into risk exposure (beta) and pricing error (alpha), and use them to predict future price.


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

Here, $$a$$ captures the characteristic-related pricing-error component, while $$B$$ maps characteristics into factor exposures. A constant can be included in $$z_{it}$$ to allow for intercepts.

# 2 Data and Sample

## 2.1 Data and sample

Market and financial data are from CSMAR; suspension records are from RESSET. The sample covers Shanghai and Shenzhen A-shares, excluding B-shares and Beijing Stock Exchange securities. ST, *ST, PT, and suspended stocks are excluded using status known before portfolio formation. The panel is unbalanced and starts in 2006.

The project combines weekly/monthly frequency, open-to-close/close-to-close returns, and announcement-date (`annodt`)/disclosure-deadline (`deadline`) information rules. The statistics below use **weekly open-to-close returns, `annodt`, all eligible stocks, and K = 10**.

After filtering and excluding unfinished periods, the full sample contains **3,022,711 stock-week observations from 5,481 stocks**, covering **1,041 weeks, 2006-W01–2026-W22**. Descriptive statistics use this sample before model-specific feature selection.

## 2.2 Data preparation

Open-to-close returns run from the first market trading day's open to the last trading day's close; close-to-close returns start at the preceding period's closing price. Prices are adjusted for corporate actions. Valid entry prices are required, and stale close-to-close starting observations are excluded. Missing terminal quotes are valued using the latest available earlier price.

Financial reports are matched to stock returns using the latest report available before portfolio formation. Under `annodt`, availability follows the actual announcement date; under `deadline`, it follows the later of the announcement date and disclosure deadline. For weekly open-to-close portfolios, financial and market characteristics use information available by the preceding week's final trading day. ST and suspension filtration uses the trading session before entry.

The characteristics comprise **37 accounting, market, and ownership characteristics**, plus lagged return. Accounting variables use consolidated statements, with income and cash-flow measures converted to trailing-twelve-month values. Market characteristics capture size, turnover, momentum, beta, idiosyncratic volatility, liquidity, and abnormal volume.

Characteristics are cross-sectionally ranked to **[−0.5, 0.5]**.

Within each period, PCA captures shared linear patterns across stock characteristics and uses a stock's observed characteristics to estimate its missing values. The imputation method follows [Xiong and Pelger (2023), *Large dimensional latent factor modeling with missing observations and applications to causal inference*, Journal of Econometrics, 233(1), 271-301](https://doi.org/10.1016/j.jeconom.2022.04.005), applied separately to each period's cross-section. This step uses **7 PCA components**, separate from the return model's K = 10 factors, and requires at least ten observed characteristics per stock-period observation. Returns and status flags are not imputed. Across the original source panel, characteristic coverage rises from **93.65% to 98.68%**. This backtest retains the previously prepared characteristic panel; models estimated on subsamples additionally re-rank characteristics within groups.

## 2.3 Descriptive statistics

The table summarizes the full sample, pooling stock-week observations with equal weights. Returns are in percent; excess returns subtract the same week's risk-free return.

| Variable | Mean (%) | Std. dev. (%) | 5th percentile (%) | Median (%) | 95th percentile (%) |
| --- | ---: | ---: | ---: | ---: | ---: |
| Stock return | 0.3430 | 6.7773 | -9.4545 | 0.0000 | 11.1989 |
| Stock excess return | 0.3078 | 6.7773 | -9.4922 | -0.0286 | 11.1623 |

## 2.4 Estimation and model evaluation

Each model uses **30 features plus an intercept**, with `LME`, `BEME`, `Beta`, and `ret_lag` fixed. Additional features are selected using Fama–MacBeth coefficient t-statistics with HAC adjustment, excluding candidates highly correlated with selected variables. The number of return factors is fixed at **K = 10**.

IS estimation uses the full eligible sample. OOS feature selection uses the first **90 usable weeks**, parameters are re-estimated using an expanding window of preceding observations.

The evaluation below uses a separately fitted **excess-return model**. Reconstruction uses realized-period factors and measures explanatory fit; prediction replaces those factors with their historical means. OOS reconstruction therefore is not an ex ante return forecast. Total R² pools squared errors relative to squared excess returns, using zero as the benchmark; time-series and cross-sectional R² average the corresponding ratios across stocks and periods. RMSE is the square root of mean squared error.

| Evaluation | Total R² (%) | Time-series R² (%) | Cross-sectional R² (%) | RMSE (percentage points) |
| --- | ---: | ---: | ---: | ---: |
| IS reconstruction | 37.5045 | 29.5103 | 28.7015 | 5.2472 |
| OOS reconstruction | 36.6250 | 34.6697 | 27.8632 | 5.2322 |
| IS mean-factor prediction | 0.2479 | 0.1203 | 0.7383 | 6.6292 |
| OOS mean-factor prediction | -0.3005 | -0.2932 | 0.1455 | 6.5822 |

The model explains a substantial share of realized return variation, while individual-stock predictive R² remains close to zero. OOS results remain conditional on the previously calibrated imputation procedure. Missing panel observations and stale-price valuation also limit interpretation as an executable trading strategy; portfolio results are discussed in Section 3.

## 2.5 Factor Rank IC and ICIR

For the weekly open-to-close, `annodt`, `all`, K = 10 excess-return model, the table reports OOS results over 947 weeks (2007-W44–2026-W22). Rank IC is the cross-sectional Spearman correlation between factor exposures and subsequent holding-period returns; ICIR is the weekly IC mean divided by its standard deviation, without annualization. Factor signs are aligned across estimation windows using historical information only.

| Factor | Mean Rank IC | ICIR |
| --- | ---: | ---: |
| F1 | 0.0335 | 0.2181 |
| F2 | -0.0113 | -0.0582 |
| F3 | 0.0187 | 0.1228 |
| F4 | 0.0124 | 0.0949 |
| F5 | 0.0517 | 0.3622 |
| F6 | 0.0204 | 0.1494 |
| F7 | 0.0358 | 0.2444 |
| F8 | 0.0099 | 0.0676 |
| F9 | -0.0254 | -0.1773 |
| F10 | 0.0001 | 0.0006 |

# 3 Trading Strategies

This section reports the project's **linear RPCA** strategies for **weekly open-to-close returns, actual announcement dates (`annodt`), all stocks (`all`), and K = 10**. IS covers **1,037 weeks, 2006-W01–2026-W22**; OOS covers **947 weeks, 2007-W44–2026-W22**, following 90 usable training periods. Feature selection is rerun on the filtered sample; OOS selection uses only that initial window, followed by expanding-window estimation.

The trading strategies use **raw-return managed portfolios**. Excess returns are used when calculating Sharpe ratios. The separately fitted excess-return model used for the R² evaluation in Section 2 does not replace the trading model here.

## 3.1 Managed portfolios and the pure-alpha strategy

Let $$r_t$$ be the vector of stock returns and $$Z_{t-1}$$ the feature(Characteristic) matrix, containing the 30 selected features and an intercept. Managed returns are

$$
g_t=Z_{t-1}^{+}r_t,
$$

where $$+$$ denotes the Moore–Penrose inverse. With a full-rank design, this equals $$(Z_{t-1}^{\top}Z_{t-1})^{-1}Z_{t-1}^{\top}r_t$$. The same regression operator is used to map every managed-portfolio direction back into stock weights.

At period t, PCA is estimated from demeaned historical managed returns. With orthonormal loading columns $$B_{t-1}$$, define

$$
a_{t-1}=(I-B_{t-1}B_{t-1}^{\top})\overline g_{t-1},
\qquad
f_t=B_{t-1}^{\top}g_t.
$$

The pure-alpha direction is $$h_{\alpha,t-1}=a_{t-1}$$, giving the unscaled payoff

$$
R^0_{\alpha,t}=a_{t-1}^{\top}g_t,
\qquad
w^0_{\alpha,t}=(Z_{t-1}^{+})^{\top}a_{t-1}.
$$

The current realized return determines the payoff, but it is not needed to determine these OOS weights. The alpha direction is orthogonal to the estimated loading space; this alone does not establish that the resulting stock portfolio is market-neutral or free of omitted risks.

## 3.2 Factor MVE strategy

Using the current training-window loadings, I constructed the history of factor-return series. Let $$\mu_{f,t-1}$$ and $$\Sigma_{f,t-1}$$ denote its sample mean and covariance. The factor mean–variance direction is

$$
q_{f,t-1}=\Sigma_{f,t-1}^{-1}\mu_{f,t-1},
\qquad
h_{\mathrm{MVE},t-1}=B_{t-1}q_{f,t-1}.
$$

Its unscaled payoff is $$R^0_{\mathrm{MVE},t}=q_{f,t-1}^{\top}f_t$$. This optimizer uses the raw-return moments of the implemented strategy and does not impose a unit-budget or fixed-gross-exposure constraint.

## 3.3 Combined Alpha-MVE strategy

The combined strategy jointly optimizes the alpha portfolio and all ten factors. For each historical training period $$s$$, form

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

Here $$\widehat\sigma_{m,t-1}$$ is the sample standard deviation (`ddof=1`) of the formation-eligible-market, formation-capitalization-weighted weekly return over the actual training window. IS uses its full-sample value; OOS uses only earlier training periods. This scaling applies to final stock weights, not to the managed returns or PCA loadings. It does **not** target a common strategy volatility, impose full investment, or cap gross leverage.

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

Normalizing each leg removes the common positive multiplier $$c_t$$, but also changes the strategy's long and short exposure magnitudes. Because those exposures vary over time, the two reported Sharpe ratios need not coincide.

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

| Strategy | IS original-weight SR | OOS original-weight SR | IS normalized long-short SR | OOS normalized long-short SR |
| --- | ---: | ---: | ---: | ---: |
| MVE | 3.8653 | 3.2078 | 3.8582 | 3.2893 |
| Alpha | 3.9517 | 3.8385 | 3.9671 | 3.8396 |
| Alpha-MVE | 5.5279 | 4.7696 | 5.5139 | 4.8328 |

The Alpha-MVE OOS Sharpe is **4.7696 for original weights and 4.8328 for normalized long-short weights**.

| OOS strategy | non_filtered SR | SR | Change |
| --- | ---: | ---: | ---: |
| MVE / long_short | 3.1492 | 3.2893 | 0.1401 |
| Alpha / long_short | 3.9020 | 3.8396 | -0.0624 |
| Alpha-MVE / long_short | 4.7671 | 4.8328 | 0.0657 |

\* `non-filtered SR` reports the results of non-filtered sample, i.e. the sample who keep ST and suspended observations.

The normalized OOS MVE, Alpha, and Alpha-MVE portfolios have mean **weekly** payoffs of **0.4258%, 0.3106%, 0.5340%**, respectively, and annualized volatility of **6.732%, 4.207%, 5.745%**.

## 3.6 Long-only strategies

I also evaluates two constrained trading strategies. `MVE-nonnegative` combines the full-market long-only portfolio with ten factors; `MVE-alpha-nonnegative` additionally includes the alpha portfolio. For the relevant training-return mean $$\mu$$, covariance $$\Sigma$$, and current stock weight mapping M, the optimizer solves

$$
\min_q\;\frac12 q^{\top}\Sigma q-\mu^{\top}q
\quad\text{subject to}\quad Mq\geq0.
$$

The constraint is on mapped **stock weights**, not on the signs of the factor coefficients. The same historical market-volatility scaling is applied afterward.

| Strategy | IS original-weight SR | OOS original-weight SR | IS normalized long-only SR | OOS normalized long-only SR |
| --- | ---: | ---: | ---: | ---: |
| MVE-nonnegative | 0.4780 | -0.1036 | 0.4874 | 0.1723 |
| MVE-alpha-nonnegative | 0.5324 | -0.0413 | 0.5261 | 0.2112 |

The constrained OOS results are much weaker in this specification. Normalization can change their Sharpe because it removes time-varying total exposure; it is not simply a fixed rescaling of the entire return series. The high unconstrained long-short Sharpe should therefore not be presented as evidence of comparable long-only performance.

These results do not by themselves establish market inefficiency or an executable arbitrage. Transaction costs, borrow fees, and real-world financing spreads are not deducted, short-sale availability is not enforced for the unconstrained portfolios, and stale terminal quotes are valuation marks rather than guaranteed execution prices.
