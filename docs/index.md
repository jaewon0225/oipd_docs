## 1. Introduction

### 1.1. What is OIPD?

[OIPD (Options Implied Probability Distribution)](https://github.com/tyrneh/options-implied-probability) is a powerful Python library designed for financial quants, researchers, and traders. It provides a comprehensive toolkit for transforming raw options market data into meaningful, forward-looking probability distributions of a security's future price.

The core mission of OIPD is to democratise access to sophisticated options analysis. By handling the complex mathematics of volatility modeling and risk-neutral density estimation, OIPD allows you to focus on what matters: generating insights, managing risk, and developing novel trading strategies.

### 1.2. Core Concepts

#### 1.2.1. Implied Volatility (IV)
Implied volatility is the market's expectation of future price movements of a security. It is the volatility value that, when plugged into an options pricing model (like Black-Scholes), yields the option's current market price. High IV suggests the market expects significant price swings, while low IV implies a period of relative calm.

#### 1.2.2. The Volatility Smile & Skew
In theory, implied volatility should be the same for all options on the same underlying with the same expiration date. In practice, this is not the case. When you plot IV against strike prices, you often see a "smile" or "skew" pattern.

*   **Smile:** IV is lowest for at-the-money (ATM) options and increases for both in-the-money (ITM) and out-of-the-money (OTM) options.
*   **Skew:** More common in equity markets, where IV for OTM puts is higher than for OTM calls. This reflects the market's perception that there is a greater risk of a large downward move (a crash) than a large upward move.

OIPD fits mathematical models to these smiles to create a continuous volatility curve. The primary model used is the **SVI (Stochastic Volatility Inspired)** model, which is known for its robust and arbitrage-free representation of the volatility smile.

#### 1.2.3. Risk-Neutral Density (RND)
The risk-neutral density is a probability distribution of the future price of an asset, derived from option prices. It represents the probabilities that a risk-neutral investor would assign to different future price levels.

A key insight from financial theory is that the RND can be derived from the second derivative of the call price function with respect to the strike price. OIPD uses the fitted volatility curve to create a smooth call price function, and then calculates its second derivative to obtain the RND. This RND is a powerful tool for:

*   **Risk Management:** Quantifying the probability of extreme price moves.
*   **Trade Idea Generation:** Identifying discrepancies between your own views and the market's implied probabilities.
*   **Product Pricing:** Valuing complex derivatives.

#### 1.2.4 Fitting Methods
* **b-spline:**
* **SVI:**


### 

### 1.3. Installation

You can install OIPD and its dependencies, including `yfinance` for data fetching, using `pip`:

```bash
pip install oipd
```

For a minimal installation without data vendor integrations (if you are providing your own data), you can use:

```bash
pip install oipd[minimal]
```

## 2. Getting Started

### 2.1. Quickstart Guide

This example shows how to get a risk-neutral probability distribution for Apple (AAPL) stock.

```python
import oipd
import yfinance as yf
from datetime import date
import pandas as pd

# 1. Fetch Data
# We use yfinance to get the option chain for a specific expiry.
ticker = yf.Ticker("AAPL")
expiry = date(2025, 12, 19)
chain = ticker.option_chain(expiry.strftime("%Y-%m-%d"))
# We'll use the call options for this example.
options_df = chain.calls

# 2. Define Market Conditions
# The MarketInputs object holds all the necessary market data.
market = oipd.MarketInputs(
    underlying_price=ticker.info["currentPrice"],
    valuation_date=pd.to_datetime(date.today()),
    expiry_date=pd.to_datetime(expiry),
    risk_free_rate=0.05,  # Use a realistic rate for your analysis
)

# 3. Fit the Volatility Curve
# The VolCurve class is the main entry point.
# We create an instance and call the fit() method.
vol_curve_estimator = oipd.VolCurve()
vol_curve_estimator.fit(options_df, market)

# 4. Derive the Probability Distribution
# The implied_distribution() method returns a Distribution object.
dist = vol_curve_estimator.implied_distribution()

# 5. Query Probabilities
# Now you can query the distribution for insights.
price_target = 250.0
prob_below = dist.prob_below(price_target)
prob_above = dist.prob_above(price_target)
prob_between = dist.prob_between(240.0, 260.0)

print(f"Probability of AAPL being below ${price_target}: {prob_below:.2%}")
print(f"Probability of AAPL being above ${price_target}: {prob_above:.2%}")
print(f"Probability of AAPL being between $240 and $260: {prob_between:.2%}")

# You can also get the expected value (mean) of the distribution.
ev = dist.expected_value()
print(f"Expected value of AAPL at expiry: ${ev:.2f}")
```

### 2.2. Fitting a Volatility Smile

The `oipd.VolCurve` class is highly configurable. You can specify the fitting method, pricing engine, and more.

```python
# Example of a more customised VolCurve
vol_curve_bs = oipd.VolCurve(
    method="svi",
    pricing_engine="bs",  # Use Black-Scholes instead of Black-76
    price_method="mid"
).fit(options_df, market)

# Accessing fitted parameters and diagnostics
print("Fitted SVI parameters:", vol_curve_bs.params)
print("ATM Volatility:", vol_curve_bs.at_money_vol)
print("Fit Diagnostics:", vol_curve_bs.diagnostics)
```

### 2.3. Deriving a Probability Distribution

The `Distribution` object gives you access to the PDF (Probability Density Function) and CDF (Cumulative Distribution Function), which you can use for plotting.

```python
import matplotlib.pyplot as plt

# Get the PDF and CDF
prices = dist.prices
pdf = dist.pdf
cdf = dist.cdf

# Plot the PDF and CDF
fig, ax1 = plt.subplots(figsize=(10, 6))

ax1.plot(prices, pdf, 'b-', label='PDF')
ax1.set_xlabel('Price')
ax1.set_ylabel('Probability Density', color='b')
ax1.tick_params('y', colors='b')

ax2 = ax1.twinx()
ax2.plot(prices, cdf, 'r-', label='CDF')
ax2.set_ylabel('Cumulative Probability', color='r')
ax2.tick_params('y', colors='r')

plt.title('Risk-Neutral PDF and CDF for AAPL')
fig.tight_layout()
plt.show()
```

## 3. User Guide

### 3.1. The `VolCurve` Interface

The `VolCurve` is the workhorse of the `oipd` library. Its constructor allows you to configure the estimation process.

`VolCurve(method='svi', method_options=None, solver='brent', pricing_engine='black76', price_method='mid', max_staleness_days=3)`

*   **`method`**: The smile fitting algorithm.
    *   `'svi'`: (Default) The SVI (Stochastic Volatility Inspired) model. Recommended for most use cases.
    *   `'bspline'`: Fits a B-spline to the implied volatility data.
*   **`method_options`**: A dictionary of options for the chosen method. For SVI, this can include `random_seed`, `max_iter`, etc.
*   **`solver`**: The numerical solver for backing out implied volatility from option prices.
    *   `'brent'`: (Default) A robust root-finding algorithm.
    *   `'newton'`: Newton's method, which can be faster but less stable.
*   **`pricing_engine`**: The options pricing model to use.
    *   `'black76'`: (Default) The Black-76 model, which prices options on futures. It uses the forward price.
    *   `'bs'`: The Black-Scholes model, which prices options on the underlying stock. It requires dividend information.
*   **`price_method`**: Which price to use from the option chain data. Can be `'mid'`, `'last'`, `'bid'`, or `'ask'`.
*   **`max_staleness_days`**: The maximum age in days for an option quote to be considered valid.

### 3.2. The `VolSurface` Interface

The `VolSurface` class allows you to work with multiple expiries at once. It takes the same configuration parameters as `VolCurve`.

```python
# 1. Fetch data for multiple expiries (this is a simplified example)
# In a real scenario, you would concatenate chains for different expiries.
multi_expiry_df = pd.concat([
    ticker.option_chain(expiry.strftime("%Y-%m-%d")).calls,
    ticker.option_chain(next_expiry.strftime("%Y-%m-%d")).calls
])


# 2. Fit the surface
vol_surface = oipd.VolSurface(expiry_column='expiry').fit(multi_expiry_df, market)

# 3. Get the distribution surface
dist_surface = vol_surface.implied_distribution()

# 4. Slice the surface for a specific expiry
expiry_to_get = pd.to_datetime(expiry)
single_dist = dist_surface.slice(expiry_to_get)
```

### 3.3. Working with Probability Distributions

The `Distribution` object provides several ways to query the probability of future price movements:

*   `prob_below(price)`: P(S < price)
*   `prob_above(price)`: P(S >= price)
*   `prob_between(low, high)`: P(low <= S <= high)
*   `expected_value()`: The mean of the distribution.
*   `variance()`: The variance of the distribution.

### 3.4. Data Sources and Market Inputs

OIPD is designed to be data-source agnostic. The `fit` method expects a pandas DataFrame with the following columns (can be remapped using the `column_mapping` parameter):

*   `strike`: The strike price.
*   `expiry`: The expiration date.
*   `option_type`: `'call'` or `'put'`.
*   `last_price`: The last traded price.
*   `bid`: The bid price.
*   `ask`: The ask price.
*   `volume`: The trading volume.
*   `open_interest`: The open interest.
*   `last_trade_date`: The date of the last trade.

The `MarketInputs` class is crucial for providing the context for the valuation:

`MarketInputs(valuation_date, underlying_price, risk_free_rate, expiry_date=None, dividend_yield=0.0, dividend_schedule=None, risk_free_rate_mode='continuous')`

### 3.5. Advanced Configuration

#### SVI Method Options
You can pass a `SVICalibrationOptions` object or a dictionary to `method_options` to control the SVI calibration:

```python
from oipd.core.vol_surface_fitting.shared.svi_types import SVICalibrationOptions

svi_options = SVICalibrationOptions(
    random_seed=42,
    max_iter=1000,
    tolerance=1e-6
)

vol_curve = oipd.VolCurve(method_options=svi_options).fit(options_df, market)
```

#### Handling Dividends
When using the `'bs'` pricing engine, you must provide dividend information. You can provide either a continuous `dividend_yield` or a `dividend_schedule` (a list of (date, amount) tuples).

### 3.6. Using Different Data Sources



## 4. API Reference

### 4.1. `oipd.interface`
This is the main entry point for users.
*   **`VolCurve`**: The primary class for fitting single-expiry volatility smiles.
*   **`VolSurface`**: For fitting multi-expiry volatility surfaces.
*   **`Distribution`**: A container for the resulting risk-neutral distribution.
*   **`DistributionSurface`**: A container for a multi-expiry distribution surface.

### 4.2. `oipd.pipelines`
This module contains the high-level stateless pipelines that orchestrate the estimation process.
*   **`vol_estimation.fit_vol_curve_internal`**: The core function that takes cleaned data and market inputs and returns a fitted volatility curve.
*   **`prob_estimation.derive_distribution_from_curve`**: The function that takes a fitted curve and derives the probability distribution.

### 4.3. `oipd.core`
This module contains the low-level algorithms and building blocks.
*   **`vol_surface_fitting`**: Contains the implementations of the smile fitting algorithms like SVI and b-spline.
*   **`probability_density_conversion`**: Includes functions for converting the call price curve to a PDF, such as `finite_diff_second_derivative`.
*   **`data_processing`**: Functions for cleaning, validating, and preparing the raw options data.

## 5. Examples

You can find more detailed examples in the `examples` directory of the project:

*   **`appl_example_new_api.py`**: A complete example of using the `VolCurve` API.
*   **`surface_example.py`**: Demonstrates how to use the `VolSurface` class to work with multiple expiries.
*   **`OIPD_colab_demo.ipynb`**: An interactive Colab notebook that walks through the library's features.

## 6. Contributing

### 6.1. How to Contribute

Contributions are welcome! If you have a bug fix, feature proposal, or improvement, please open an issue on GitHub to discuss it first. Then, you can submit a pull request.

### 6.2. Development Setup

1.  Clone the repository:
    ```bash
    git clone https://github.com/tyrneh/options-implied-probability.git
    cd options-implied-probability
    ```
2.  Install the library in editable mode with development dependencies:
    ```bash
    pip install -e .[dev]
    ```

### 6.3. Code Style and Conventions

We use `black` for code formatting and `isort` for organising imports. Before submitting a pull request, please run these tools:

```bash
black .
isort .
```

We also use `mypy` for static type checking. Please ensure your code passes the type checks.
