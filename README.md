# Laspeyres index: weight-sensitivity sweep

`laspeyres_weight_sensitivity.py` shows which spending category's weight a household Laspeyres price index is most sensitive to. For each category in turn, it sets that category's weight to every value from 0% to 50%, rescales the other categories so they keep their original proportions, and recomputes the index. All the sweeps go on one chart, and the steepest line is the category whose weight matters most.

## Requirements

Python 3 with numpy, pandas and matplotlib 3.5 or later; no other packages. Tested with Python 3.12.3, numpy 2.4.4, pandas 3.0.2 and matplotlib 3.10.8, with every warning raised as an error.

## Input

The script reads `household_basket`, a list of dicts with one entry per category.

| Key | Meaning |
|---|---|
| `Item` | Category name; must be unique |
| `Price_2019` | Base-period price or price-index level; greater than 0 |
| `Price_2024` | Comparison-period price in the same units; greater than 0 |
| `weight` | 2019 expenditure share; 0 or more. Rescaled to sum to 1 if it doesn't already (e.g. percentages) |

```python
household_basket = [
    {"Item": "Housing", "Price_2019": 100.0, "Price_2024": 124.0, "weight": 0.33},
    {"Item": "Food",    "Price_2019": 100.0, "Price_2024": 126.0, "weight": 0.14},
    # ... one dict per category (illustrative values)
]
```

`weight` must be an expenditure share, not a quantity: with share weights, the Laspeyres index is the share-weighted mean of the price relatives. To convert quantities, use `weight = Price_2019 * quantity / sum(Price_2019 * quantity)`. As a check, the base index the script prints should equal the index you already have.

## Running it

From a notebook or your own script, in the same folder as the file:

```python
import matplotlib.pyplot as plt
from laspeyres_weight_sensitivity import run_weight_sensitivity

basket, sweep, summary, fig = run_weight_sensitivity(household_basket)
plt.show()  # only needed outside a notebook
```

Importing the file does not run the analysis; calling `run_weight_sensitivity` does. Pass `save_outputs=False` to skip writing files.

In a notebook, you can instead run the whole file in the notebook's namespace, so it can see `household_basket`:

```python
%run -i laspeyres_weight_sensitivity.py
```

This also leaves `basket`, `sweep`, `summary` and `fig` in the notebook.

As a standalone script, add an import for your data near the top of the file (for example `from basket_data import household_basket`), then run:

```bash
python laspeyres_weight_sensitivity.py
```

## Outputs

The script prints the base index, the most weight-sensitive category, the largest contributor to the change, and a summary table ordered from most to least sensitive. It adds a note if the weights did not sum to 1, or if a category's actual weight is above `SWEEP_MAX` (its dot then falls outside the chart).

| Column | Meaning |
|---|---|
| `weight_pct` | Actual weight, % of the basket |
| `price_change_pct` | Price change from 2019 to 2024, % |
| `index_excl_item` | Index of the other categories (2019 = 100); the line's value at 0% weight |
| `slope_pts_per_pp` | Index points per percentage point of weight; the sign gives the direction |
| `contribution_pts` | Index points of the total change due to the category; sums to index − 100 |
| `share_of_change_pct` | Contribution as % of the total change |

It writes three files to the working directory, overwriting earlier runs: `laspeyres_weight_sensitivity.png` (the chart, 200 dpi), `laspeyres_weight_sensitivity_sweep.csv` (the index at every swept weight, one column per category) and `laspeyres_weight_sensitivity_summary.csv` (the table above).

On the chart, each line is one category's sweep. The most sensitive category is drawn thicker, and the legend is ordered by sensitivity and shows each slope. The dots mark the actual weights, where every line crosses the dashed line at the base index.

## Method

```
R_i           = Price_2024_i / Price_2019_i
index         = 100 * sum_i(w_i * R_i),  with sum_i(w_i) = 1
sweep for k   : w_k' = t,  w_j' = w_j * (1 - t) / (1 - w_k) for j != k,  t = 0, 0.01, ..., 0.50
result        : index_k(t) = 100 * (t * R_k + (1 - t) * R_rest_k)
slope         : R_k - R_rest_k  (index points per percentage point of weight)
```

`R_rest_k` is the weighted price relative of the other categories. Rescaling keeps the weights summing to 1, so the index remains the cost of a fixed basket instead of growing with the basket's size. It also keeps the other categories' mix with each other unchanged, so each sweep moves one category against the rest. As a result, the index is linear in the swept weight, and the slope depends on how far the category's price change is from the rest of the basket, not on the size of its weight.

The script stops with an error if a swept weight vector does not sum to 1, if the slope measured from the recomputed sweep differs from `R_k - R_rest_k`, or if the contributions do not sum to the total change. It also rejects missing keys, duplicate items, non-positive prices, negative or all-zero weights, and a basket where one category holds all the weight.

## Reading the result

The largest contributor and the most sensitive category answer different questions, and they can be different categories. `share_of_change_pct` shows what drove the change. `slope_pts_per_pp` shows which weight would move the result most if it were wrong: an error of e percentage points in that weight shifts the index by slope × e points. If that shift is too small to change any decision, the result is robust to the weighting.

The index uses a fixed 2019 basket. It ignores households shifting spending toward items that became relatively cheaper, so it tends to overstate the cost-of-living increase, and most where price changes diverge. Those are also the categories with steep lines.

Ranking by `slope_pts_per_pp` treats one percentage point of weight as equally uncertain for every category. If weight errors are more likely to be proportional to the share, rank by `slope_pts_per_pp * weight_pct` instead.

Each sweep is one-at-a-time: one category moves against all the others in fixed proportion. Joint changes, such as housing rising while transport falls, are not covered.

## Configuration

Edit `SWEEP_MIN`, `SWEEP_MAX` and `SWEEP_STEPS` at the top of the file to change the grid (defaults `0.0`, `0.5` and `51`, which gives 1-percentage-point steps), and `OUTPUT_STEM` to rename the output files. The steps can also be called separately: `prepare_basket`, `sweep_weights`, `summarise` and `plot_sensitivity`. `run_weight_sensitivity` chains them and returns `(basket, sweep, summary, fig)`.
