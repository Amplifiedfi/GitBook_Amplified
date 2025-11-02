---
description: How AI Agents choose the strategies for Amplified AI Vault
icon: vault
cover: ../../.gitbook/assets/image.avif
coverY: 0
---

# AI Agents Portfolio Engine

Below is a comprehensive overview of how our Amplified AI engine chooses the best DeFi yield farming strategy for a given asset (ETH, BTC, or USD).&#x20;

## Agenda:

1\. Data Ingestion

2\. Strategy Generation

3\. Metric Calculation

4\. Scoring and Strategy Selection

5\. Why Different Weights Matter

6\. How AI Agents Choose the Final Strategy

## 1. Data Ingestion

### 1.1 Allowed Tokens

We create a dictionary of whitelisted tokens that can be considered for each asset category (ETH, BTC, or USD). For example, in the snippet for ETH, we allow different types of wrapped or liquid-staked versions of ETH:

```
WHITELISTED_TOKENS = {
    "ETH": {
        "wETH": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
        "weETH": "0xCd5fE23C85820F7B72D0926FC9b05b43E359b7ee",
        "rsETH": "0xA1290d69c65A6Fe4DF752f95823fae25cB99e5A7",
        ...
    },
    "BTC": {
        "wBTC": "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599",
        ...
    },
    "USD": {
        "USDC": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "USDT": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        ...
    }
}
```

We do this to ensure that when we look for pools, we only focus on pools offering yields for the tokens we explicitly trust or support in Amplified Protocol. This is especially critical for protocol security and standardization of yield calculations.

### 1.2 Fetching Pool Data

Our backend integrates with different on-chain and off-chain price, apy, liquidity data sources that:

## 2. Strategy Generation

Once we have valid pools, the StrategyService creates a series of potential strategies. Each strategy is essentially a subset of pools that match specific criteria. The service does this by:

1\. Filtering by allowed tokens

2\. Defining multiple “profiles” (or “templates”) of strategies with different constraints. Examples include:

> • Ultra Safe: Zero impermanent loss (IL) risk, higher TVL, must not be trending down.
>
> • Conservative Growth: Small IL risk allowed, moderate TVL, must not be trending down.
>
> • Balanced Score or Balanced APY: Slightly relaxed IL risk constraints, moderate TVL thresholds.
>
> • Growth or Aggressive Growth: Higher IL tolerance, smaller minimum TVL constraints.
>
> • Blue Chip, Maximum Yield, High Risk, etc.

**Each strategy template combines criteria like:**

• Impermanent Loss (IL) threshold

• Whether the pool APY is trending down

• Minimum Total Value Locked

• Sorting preference, e.g., by score, apy, etc.

• Number of pools to select (e.g., 2, 3, 4, 5, 6 pools)

As a result, the system can generate multiple candidate strategies from the same set of pools but for different risk profiles. Each strategy is then passed on to the MetricsService for in-depth financial analysis.

## 3. Metric Calculation

The Metrics Service is responsible for computing advanced financial metrics for each generated strategy. This is where historical data for each pool turned into performance metrics.

### 3.1 Pulling Historical Pool Data

For every pool in the strategy, we do the following:

1\. Fetch daily historical yields.

2\. Extract the APY and compute daily return est.:

$$
(\sqrt[365]{1 + \text{APY decimal}} - 1
$$

3\. Resample the data to a daily frequency to ensure consistent timelines across pools.

### 3.2 Aggregating Returns & Weighting

If a strategy has multiple pools, we assume equal weights across the pools by default (though this can be adjusted). We take each pool’s daily returns, multiply by its weight, and then sum them to get a daily return time series for the entire strategy.

## 3.3 Financial Metrics

The time series of daily returns allows us to calculate:

1\. Annualized Return:&#x20;

$$
(\bigl(1 + \text{mean\_daily}\bigr)^{365} - 1)
$$

2\. Annualized Volatility:

$$
\sigma_{\text{daily}} \times \sqrt{365}
$$

3\. Sharpe Ratio:&#x20;

$$
\frac{(\text{mean_daily} - \text{risk_free_rate_daily})}{\sigma{\text{daily}}} \times \sqrt{365}
$$

4\. Sortino Ratio: Similar to Sharpe but only penalizes downside volatility.

5\. Max Drawdown: Measures the worst peak-to-trough decline in cumulative returns.

6\. Value at Risk (VaR) & Conditional VaR: Risk measures for tail events.

7\. Calmar Ratio:

$$
\frac{\text{Annual Return}}{\text{Max Drawdown}}
$$

8\. Alpha / Beta

9\. Skewness & Kurtosis: For distribution shape of returns.

10\. Omega Ratio: Another performance measure comparing returns above a threshold (e.g., risk-free rate) to those below it.

**We also compute simpler aggregates like:**

• Weighted APY (based on pool TVL)

• Total TVL of the strategy

• Number of pools

## 4. Scoring and Strategy Selection

After we compute all metrics for every candidate strategy, we need to choose the best one. The selection uses a weighted scoring system defined in the select\_best\_strategy function:

1\. APY

* &#x20;Given the highest priority weight (70%). The rationale is that yield farming is largely about getting the best returns.

2\. Sharpe Ratio

* Weighted at 15%. Sharpe Ratio adjusts returns by their volatility. A higher Sharpe Ratio means higher risk-adjusted returns.

3\. Alpha

* Weighted at 10%. Measures the strategy’s excess return compared to a baseline or risk-free rate, capturing how well it performs above the market.

4\. Beta

* Weighted at 5%. We prefer a Beta close to 1 (neither too high nor too low).

5\. Volatility

* Weighted negatively at -10%. Higher volatility reduces the final score, punishing strategies that have large swings.

The system picks the strategy with the highest “total\_score”. This balanced formula ensures strategies that offer strong APY and robust risk-adjusted metrics come out on top.

## 5. Why Different Weights Matter

### 5.1 Emphasizing Returns vs. Risk

• High APY Weight (70%): DeFi yield-seekers are often primarily interested in maximizing returns; hence APY gets the biggest chunk of the weighting.

• Moderate Sharpe Weight (15%): Encourages stable, low-volatility returns.

• Positive Alpha Weight (10%): Rewards strategies that outperform baseline expectations.

• Beta & Volatility: Fine-tuning for risk alignment. We penalize strategies with extremely high volatility or Beta far from 1.

### 5.2 Adjusting These Weights for Different Market Conditions

In practice, these weights are configurable. For instance, during a bull market, we might reduce the penalty on volatility to chase higher yields. Conversely, in a more uncertain market, we might increase the weight of Sharpe or reduce the weight on Beta to favor safety.

## 6. How the AI Chooses the Final Strategy

Once each strategy has:

1\. Generated its set of pools

2\. Calculated metrics

3\. Received a final total\_score.

> The AI system of Agents will pick the highest-scoring strategy based on its pre-trained LLM model for decision making, based on the best asset management practises, historical performance of pools with an additional data about assets and chosen DeFi protocols.

**The final output includes:**

• Chosen strategy details: Pools, protocols, APY, risk metrics, etc.

• Generated textual description: Explaining why it is optimal and its reasoning.

• Optional visualization: A graph or flowchart of the strategy for easy interpretation.

**This approach ensures:**

• Clear, data-driven rationale for why a strategy was chosen.

• User-friendliness by providing an immediate summary and visual overview.

• Adaptability to expand or adjust if the user or the market environment changes (simply re-run the pipeline with updated or new data).

### Putting It All Together

1\. Fetch and filter DeFi pools for the specified token (ETH, BTC, USD).

2\. Generate multiple strategy candidates with different risk/reward profiles.

3\. Compute advanced time-series metrics like annual returns, Sharpe, max drawdown, etc.

4\. Score each strategy with a weighted formula focusing on returns and risk.

5\. Select the best-scoring strategy.

6\. Generate an AI description, reasoning and visualization.

Amplified AI thus delivers an optimal yield farming strategy by blending quantitative risk-return metrics with a flexible weighting scheme. It provides not just the “what” but also the “why,” offering transparency in how it arrives at the final decision.

## Conclusion

Our Amplified AI system is designed to filter, score, and select DeFi yield farming strategies in a manner that balances high returns with acceptable risk. By leveraging on-chain and off-chain data, computing advanced metrics, and weighting them in a coherent scoring system, the AI outputs a final recommended strategy aligned with the user’s risk appetite and yield goals. The architecture is modular, so it can be adapted for different market conditions or updated with new assets and new weighting preferences as needed.
