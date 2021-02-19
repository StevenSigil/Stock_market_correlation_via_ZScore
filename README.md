# Z-score based trading strategy

*Note: The notebook used in this project contains the script used for backtesting. This is meant for reference as in the current state, the notebook is not setup to use the dependancies.*

## Introduction
It seems likely that two ETF's in the same market offering the same service should have a correlation of market prices.  For this example, I have selected American Airlines (AAL) and United Airlines (UAL) to be compared. Can we watch one of these ETF's change in value and capitalize on their correlated counterpart?


### Hypothesis
1. Are these two assets correlated
2. If correlated to a reasonable level, can we assume a trend in one asset based on recent changes in price of the other?

## Testing

I have decided to use data from years passed to avoid any current day bias that may be found in these ETF's.  This will also allow a check on the data from other peoples examples.  The time frame selected is January 1st, 2015 - January 1st, 2017.  Data was gathered from Yahoo Finance's historical prices for both assets.

### 1. Finding Correlation

After cleaning up the data (AAL had an additional row to UAL - Last row dropped), we see a correlation of 91.26%. A higher correlation would be ideal, however for testing purposes, this will suffice.
```py
np.corrcoef(american['High'], united['High'])

"""
RESULTS

array([[1.        , 0.91261514],
       [0.91261514, 1.        ]])
"""
```

![AAL & UAL 2015-2017](https://github.com/StevenSigil/testing_finances/blob/main/zscore_mavg_diff/charts/correlation_checking_AAL_UAL.png)

### 2. Finding the spread differential
The diffential here is the main determinate of what action to take. When the result (after normalizing via a zscore) is > 1 standard deviation, we are to sell AAL. Same goes for the reverse.

```py
# Spread differential

spread = american['Price'] - united['Price']
spread.plot(label='Spread', figsize=(12,8), title='Spread Difference - AAL vs UAL')
plt.axhline(spread.mean(), c='g', label='average')
plt.legend()
```

![AAL & UAL Spread Differential plot](https://github.com/StevenSigil/testing_finances/blob/main/zscore_mavg_diff/charts/spread_difference_AAL_UAL.png)

### 3. Calculating a rolling Z-score and finding the standard deviation

Since the intent is to test the theory on data, as if it were live, it is necessary to use a rolling score to interpret the data as if it were live and unknown.

```py
# Calculating a rolling zscore

spread_ma1 = spread.rolling(1).mean()
spread_ma30 = spread.rolling(30).mean()
std_30 = spread.rolling(30).std()

zscore_30_1 = (spread_ma1 - spread_ma30) / std_30

"""
RESULT

Date
2015-01-02         NaN
2015-01-05         NaN
2015-01-06         NaN
2015-01-07         NaN
2015-01-08         NaN
                ...   
2016-12-23   -1.176253
2016-12-27   -1.086287
2016-12-28   -0.954296
2016-12-29   -0.994798
2016-12-30   -1.043309
Name: Price, Length: 505, dtype: float64
"""
```

Plotting the results to check the +/- 1std is sufficient

![Spread difference after calculating a rolling score](https://github.com/StevenSigil/testing_finances/blob/main/zscore_mavg_diff/charts/rolling_30d_zscore.png)

## 4. Backtest data to prove/disprove original hypothesis

To test the theory, I decided to use [Blueshift](https://blueshift.quantinsti.com/) - the supposed replacement for Quantopian's public platform.  Like Quantopian, they use the [zipline](https://www.zipline.io/) package (some of the package) to perform analysis/backtesting.

Below is the script I came up with to test the theory. It follows the same basic setup as the above data analysis before defining the trading logic.

```py
"""
Strategy based on the 30D rolling ZScore
"""

# Zipline
from zipline.api import(    symbol,
                            order_target_percent,
                            schedule_function,
                            date_rules,
                            time_rules,
                       )
import numpy as np
import pandas as pd


# Initialize - Schedule funciton 
def initialize(context):
    schedule_function(check_pairs, date_rules.every_day(), time_rules.market_close(minutes=60))

    context.aal = symbol('AAL')
    context.ual = symbol('UAL')

    context.long_on_spread = False
    context.shorting_spread = False

# Check pairs
def check_pairs(context, data):

    aal = context.aal
    ual = context.ual

    prices = data.history([aal, ual], 'price', 30, '1d')
    short_prices = prices.iloc[-1:]

    # Spread
    ma_30 = np.mean(prices[aal] - prices[ual])
    std_30 = np.std(prices[aal] - prices[ual])

    ma_1 = np.mean(short_prices[aal] - short_prices[ual])

    if std_30 > 0:
        zscore = (ma_1 - ma_30)/std_30

        if zscore < 1.0 and not context.shorting_spread:
            # Short UAL - Long AAL
            try:
                order_target_percent(aal, -0.5)
                order_target_percent(ual, 0.5)
                context.shorting_spread = True
                context.long_on_spread = False
            except:
                pass
        elif zscore > 1.0 and not context.long_on_spread:
            # Short AAL - Long UAL
            try:
                order_target_percent(aal, 0.5)
                order_target_percent(ual, -0.5)
                context.shorting_spread = False
                context.long_on_spread = True
            except:
                pass
        elif abs(zscore) < 0.1:
            # Exit positions
            order_target_percent(aal, 0)
            order_target_percent(ual, 0)
            context.shorting_spread = False
            context.long_on_spread = False

```

## Results

The following is the outcome of the aforementioned potential strategy compared to the Benchmark results provided by Blueshift.  This was set up to run through the same dates as described above, with a $10,000 starting capitol.

![Results of theory compared to Blueshift's benchmark](https://github.com/StevenSigil/testing_finances/blob/main/zscore_mavg_diff/charts/zscore%20results.png)

As you can see, the strategy did not work as expected. Yielding a return of -6.62% on the original investment.  

The biggest cause for this difference appears to happen tword the latter half of the test. Looking back at the first plot of the prices for both assets, you can see there is a massive difference in price. 

At the end, it seems the results aren't conclusive to proving/disproving the strategy. Instead, what we did find out, is a correlation of less-than 92% is not adequate for this theory.