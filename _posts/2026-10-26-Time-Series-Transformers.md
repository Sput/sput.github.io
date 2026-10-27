# Forecasting Store Sales with a Temporal Fusion Transformer

*This blog post used AI to assist with writing. First three paragraphs are all me, last three is about 50% me, 50% AI written*


We all know transformers can be used for language models, its pretty much all we ever hear about right now. This is the first in my series of tests to see "Is there anything a Transformer can't do". My first attempt is to use a Transformer to predict time-series data. For this experiment I'm using this Kaggle competition: https://www.kaggle.com/competitions/store-sales-time-series-forecasting/overview. This dataset asks a model to predict how much of each product family a store will sell over the next 16 days. The challenge is that thousands daily series datasets must be forecast together: stores differ in size and location, product families behave differently, promotions and holidays shift demand, and sales repeat on weekly and yearly cycles. This project explores a central idea: Transformers, best known for analyzing language, can also analyze time series by learning which past moments and signals matter for a forecast.

![Store data](/images/store_data.png)

# This part should be easy for non-technical readers to understand

The initial paper on Transformers was titles "Attention is all you need", the central idea being that a nueral network that works with language data should know what words (tokens) other words (tokens) should pay **ATTENTION** to. Here we are using a transformer as a forecaster that can look back over many days and pay more **ATTENTION** to the ones that seem relevant—not just the most recent day. For example, it may find that the same weekday last week, a similar seasonal period last year, or a recent promotion is useful context. This attention mechanism helps the model relate events separated in time and combine those clues when predicting the next 16 days.

Rather than building a separate model for every product at every store, the solution trains one shared model across all of them. The Transformer can learn patterns that recur across stores and product families, while also using each store’s and product’s identity to account for differences. It receives clues about the future that are already known, such as calendar dates, holidays, and planned promotions, as well as lagged records of past sales. Its predictions are combined with those from an established seasonal forecasting approach, like taking a weighted compromise between two forecasters; the best-scoring tested candidate used a 60% share from the Transformer model.

![transformer](/images/transformer.png)

# Definitions


 ┌─────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ Term        │ Meaning here                                                                                                    │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Lag         │ A value from an earlier date in the same store–product series, used as a predictor for today’s sales. For       │
 │             │ example, a 21-day sales lag for August 16 uses that series’ sales from July 26. The model also uses seasonal    │
 │             │ lags such as 364, 365, and 730 days.                                                                            │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Input       │ The continuous span of recent history the model reads as context for a forecast. The full v3/v4 runs use a      │
 │ window      │ 90-day input window. Separate lag covariates can reach further back than those 90 days.                         │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Batch size  │ How many sampled training examples (forecast windows) are processed together for one training update. The full  │
 │             │ runs use 256. This does not mean 256 different stores or series.                                                │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Stride      │ The gap between sampled training-window positions along a time series. A stride of 14 samples windows 14 days   │
 │             │ apart; it reduces overlapping examples and training work. It does not change the 16-day forecast horizon.       │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Epoch       │ One pass through the sampled training examples. Five epochs means up to five passes; early stopping can end     │
 │             │ training sooner.                                                                                                │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Patience    │ How many validation epochs to wait without sufficient improvement before early stopping. Full runs use 2, with  │
 │             │ val_loss as the monitored quantity and a minimum improvement threshold. It only affects training when           │
 │             │ validation and early stopping are enabled.                                                                      │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Covariate   │ An input feature that may help predict sales, beyond the sales value being forecast. Examples include promotion │
 │             │ counts, calendar and holiday indicators, oil prices, store/product identity, and safely lagged sales or         │
 │             │ transaction data. Some are known in advance (calendar, planned promotions); others are historical-only and must │
 │             │ be lagged to avoid using unknown future values.                                                                 │
 ├─────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ Log         │ The promotion-count feature transformed as log1p(onpromotion), meaning ln(1 + onpromotion). Zero remains zero,  │
 │ promotion   │ while large counts are compressed. It is a transformation of the promotion input—not the sales target.          │
 └─────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘


# This part gets a little (but not a lot) more technical

The name for a time series oriented transformer is: Temporal Fusion Transformer (TFT). This TFT was implemented with Darts and trained across a dataset of 1,782 store-family series'. TFT combines recurrent sequence processing, attention over temporal representations, variable-selection components, and gating. **Attention** lets the model form context-dependent relationships across the input history, while the recurrent and gating components support local temporal structure and selective information flow. Each series’ target is transformed and standardized using statistics fitted only on the dates available. Static categorical embeddings represent store number, family, city, state, store type, and cluster. Future covariates include log promotions, cyclic day-of-week and day-of-year terms, trend, holiday flags, scaled oil prices, and exact-date sales lags of 21, 28, 35, 42, 364, 365, and 730 days, (these specific numbers were selected by brute-forcing to find the best fits) with indicators for whether each lag is available.

![Model summary](/images/model_summary.png)

For the purposes of evaluation I used three sequential, date-blocked validation folds (three sets of date ranges), each covering 16 days. Within each fold, an inner 16-day tail supports early stopping (when improvements stop), while the outer 16-day block is reserved for scoring. The full-run configuration uses an input window of 90 days, output horizon of 16 days, batch size 256, stride 14, a maximum of 64 samples per series, and up to five epochs with patience two. The transaction-feature model adds store-level transaction-count lags at the same 21-day-or-longer offsets, standardized using fold-local history; because every lag exceeds the horizon, it does not use unknown future transaction counts.

The incumbent and TFT forecasts are blended in log space, with the blend weight selected or constrained using out-of-fold predictions rather than final submission labels. I started with 50% blend and just randomly tried different combinations until I found a 68%-TFT global blend using the transaction-feature was the best performer. 

# Result:

As of this blog post going up, I'm ranked #123 out of 589 entrants.
