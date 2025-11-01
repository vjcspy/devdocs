# Stock price prediction



## Overview

### Requirements

This module is for building components for a stock futures contract prediction application.

The goal of this application is to predict future contract prices with a specific success criterion.

### Data

We have some kind of stock data

#### Order flow

```
# timestamp, volume, price, action(sell/buy)
[[1750924181, 1000, 10600, "B"], [1750923255, 700, 10500, "S"],…]
```

#### Stock price by day

```json
{
            "date": "2025-08-04",
            "high": 33700,
            "low": 32400,
            "close": 33450,
            "open": 32500,
            "volume": 47826801,
            "fBuyVal": 65973, // foregin buy value
            "fBuyQty": 2009752, // foregin buy qty
            "fSellVal": 474288, // foregin sell value
            "fSellQty": 14448405, // foregin sell
            "value": 1569979
}
```



## Plan

**Feature Engineering:** This involves building the features for your model.

**Data Labeling:** You need to label your data so the model knows what to predict.

**AI Model Selection:** Choose an appropriate AI model for the task.

**Training and Evaluation:** Train the model and then evaluate its performance.

**Backtesting:** Test the model on historical data to see how it would have performed in the past.

