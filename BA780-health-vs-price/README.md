# Health vs. Price: How to Be a Smart Grocery Shopper

**What Open Food Facts and Open Prices reveal about U.S. packaged groceries**

---

## Overview

Many shoppers on a budget believe that eating well costs more. This project tests that belief with open data. We matched **988,445 U.S. products** from Open Food Facts with **39,725 shelf prices** from Open Prices. We then compared prices **per 100 g within the same aisle (food group)**, because that is how a shopper actually chooses between products.

**Main finding:** in the stores we could measure, healthy food is not what costs more. Shoppers overpay for brands, packaging, and energy-dense treats, not for nutrition.

| Finding | Result |
|---|---|
| Nutri-Score A/B vs D/E, same aisle, per 100 g | A/B costs **0.83×** (95% CI 0.79–0.88) |
| Healthiest aisles (legumes, vegetables, milk & yogurt, cereals, fruit) | About **$1 per 100 g**, vs $2.30–$6 for chocolate, biscuits, nuts, cheese |
| Store brands vs national brands, same aisle, per 100 g | **0.58×** (95% CI 0.54–0.61), and healthier on average |
| Same barcode, cheapest vs priciest store | Median gap **1.14×**. For a quarter of products the gap is 1.29× or more. |
| Low-impact (Green-Score A/B) vs high-impact (D–F), same aisle | **0.93×** the price per 100 g, and +18 pp more likely to be Nutri-Score A/B |
| Swap-engine prototype | Finds a healthier, cheaper, similar swap for **89%** of D/E products |

> **Scope:** 85% of the price observations come from California, mostly Bay Area stores, and 93% come from a single contributor. The price results describe those stores. They are not national price estimates (see [Limitations](#limitations)).

---

## Repository structure

```
.
├── README.md
├── B02-00-Data-Pull.ipynb                               # Step 1: builds the input files from source
├── B02-01-Health-Vs-Price-US-Packaged-Groceries.ipynb   # Step 2: cleaning, analysis, figures, conclusions
└── input_data/
    ├── off_products_us.csv        # ~1M U.S. products: scores, nutrients, package size, labels (used by analysis)
    ├── off_products_us.parquet    # Same table with native list types (not used by analysis)
    ├── off_prices_us.csv          # U.S. Open Prices observations, 48 columns
    ├── food_groups.json           # Open Food Facts food-group taxonomy (used by analysis)
    └── categories.json            # Open Food Facts category taxonomy (downloaded; not used by analysis)
```

---

## Data sources

All sources are open data under the **Open Database License (ODbL) 1.0**.

| Source | What we use | Link |
|---|---|---|
| Open Food Facts product database | `food.parquet` (~7.9 GB), filtered to U.S. products | [huggingface.co/datasets/openfoodfacts/product-database](https://huggingface.co/datasets/openfoodfacts/product-database) |
| Open Prices | Crowd-sourced shelf prices, filtered to U.S. stores | [huggingface.co/datasets/openfoodfacts/open-prices](https://huggingface.co/datasets/openfoodfacts/open-prices) |
| Open Food Facts taxonomies | `food_groups.json`, `categories.json` | [static.openfoodfacts.org/data/taxonomies/](https://static.openfoodfacts.org/data/taxonomies/) |

### Key fields in `off_products_us.csv`

| Field | Description |
|---|---|
| `code` | Barcode (stored as text; padded to 13 digits during cleaning) |
| `product_name`, `brands` | Product name and brands, flattened from multilingual fields |
| `food_groups_tags`, `categories_tags` | Official food group and category tags, `\|`-separated |
| `nutriscore_grade` | Nutri-Score A (healthiest) to E |
| `nova_group` | Processing level, 1 (unprocessed) to 4 (ultra-processed) |
| `environmental_score_grade` | Green-Score (formerly Eco-Score), A+ to F |
| `energy-kcal_100g` … `salt_100g` | 9 nutrients per 100 g |
| `size_g_ml` | Package size in grams or milliliters, from `product_quantity` or parsed from the `quantity` text |
| `is_organic` | From official label tags (`en:organic`, `en:usda-organic`, `en:eu-organic`) |
| `in_us_catalog`, `has_us_price` | Whether the product is tagged as sold in the U.S. / has a U.S. price |

---

## How to run

The notebooks are designed for **Google Colab** and read and write files in a Google Drive folder.

### 1. (Optional) Rebuild the input data: `B02-00-Data-Pull.ipynb`

Skip this step if `input_data/` is already populated.

1. Open the notebook in Colab and choose **Runtime → Run all**.
2. When asked, mount Google Drive. Output goes to `/content/drive/MyDrive/BA780 Team 2/Codebase/input_data`. To change it, set the `OFF_DATA_DIR` environment variable or edit `default_dir`.
3. The notebook downloads the 7.9 GB product file to Colab's temporary disk once (a few minutes). DuckDB then reads only the columns it needs, so memory use stays low.
4. *Optional:* add a free Hugging Face token as a Colab secret named `HF_TOKEN`. This raises Hugging Face's rate limits and helps avoid `HTTP 429` errors.

What it does:
- **Prices:** keeps all Open Prices rows where `location_osm_address_country_code = 'US'`.
- **Products:** keeps U.S.-tagged products **plus** any barcode with a U.S. price, even when the product isn't tagged as U.S. When a barcode is duplicated, it keeps the most recently edited row.
- **Package size:** converts sizes to g/ml (oz, lb, fl oz, L, gal, and so on), handles multipacks ("6 x 1.5 oz"), and sets sizes outside 1 g–25 kg to null.
- **Taxonomies:** downloads `categories.json` and `food_groups.json`.
- Runs sanity checks on coverage (Nutri-Score, NOVA, Green-Score, package size, food group).

> Open Prices is refreshed daily, so re-running the pull will give slightly different row counts from those reported below.

### 2. Run the analysis: `B02-01-Health-Vs-Price-US-Packaged-Groceries.ipynb`

1. Open in Colab and **Run all**. The notebook mounts Drive and reads from:
   ```python
   path = "/content/drive/MyDrive/BA780 Team 2/Codebase/input_data/"
   ```
2. To run locally, remove the `google.colab` cell and set `path` to your local `input_data/` folder.

### Requirements

- Python 3.11
- `pandas`, `numpy`, `scipy` (≥ 1.11, for `ttest_ind(...).confidence_interval()` and `false_discovery_control`), `matplotlib`, `seaborn`
- Data pull only: `duckdb`, `huggingface_hub`, `pyarrow`

```bash
pip install pandas numpy "scipy>=1.11" matplotlib seaborn duckdb huggingface_hub pyarrow
```

---

## Methodology

### Cleaning

**Products**
- Barcodes are padded to 13 digits, then de-duplicated.
- Placeholder values ("unknown", "not-applicable") become missing, and Green-Score A+ is treated as A.
- Impossible nutrient values become missing (below 0, above 100 g per 100 g, above 900 kcal).
- Each product gets an official food-group name from `food_groups.json`. Level 2 is used when it exists, otherwise level 1.
- Only package sizes from **20 g to 5 kg** are kept.

**Prices**
- Only barcode prices (`type = PRODUCT`) in USD are kept.
- Duplicate price tags are removed.
- For discounted items, the regular (pre-discount) price is used.
- Only prices from **$0.10 to $100** are kept.

**Join**
- Products and prices are joined on barcode, using each barcode's median regular price. 86% of priced barcodes match a product.
- Price per 100 g = price ÷ `size_g_ml` × 100. Only values from **$0.05 to $50** are kept.

### Data funnel

| Stage | Rows |
|---|---:|
| Products file (v2 pull) | 988,447 |
| Clean product catalog | 988,445 |
|   with an official food group | 290,927 |
|   with a Nutri-Score | 384,579 |
|   with a package size (20 g–5 kg) | 179,715 |
| Aisle analysis set (Themes 1–2): foods in groups with 2,000+ products | 234,005 |
| Price observations (raw) | 42,822 |
| Clean price observations | 39,725 (26,573 barcodes) |
| Priced products in the catalog | 22,882 |
|   with price per 100 g | 9,858 |
| **Price analysis set (Themes 3–5):** foods with price per 100 g + food group | **5,752** |

### Statistical approach

- **Within-aisle comparisons.** Every price comparison is made inside a food group, or uses a *price index*: a product's log price minus the mean log price of its food group. This stops differences in product mix from driving the results.
- **Price ratios.** Prices are right-skewed, so groups are compared with the ratio of geometric means. The 95% confidence interval comes from Welch's t-test on log prices (`price_ratio()` helper).
- **Multiple testing.** When the same test is run for many food groups, p-values are adjusted with Benjamini–Hochberg FDR.
- **Other tests.** Spearman correlations across food groups, R² of group means (aisle vs brand), and a store × food-group fixed-effects regression with robust standard errors for promotions.

### Hypotheses and results

| Theme | # | Hypothesis | Result |
|---|---|---|---|
| 1. Know your aisle | H1 | The aisle explains more of the Nutri-Score than the brand | **Supported.** R² 0.40 (aisle) vs 0.29 (brand), 660 brands, 85,172 products |
| | H2 | More ultra-processed aisles have fewer A/B products | **Supported.** Spearman ρ = −0.77 across 22 food groups |
| 2. The greener choice | H3 | In the same aisle, low-impact products are more often A/B | **Mostly supported.** +17.6 pp on average, positive in 7 of 9 groups (reverses for rice/grains) |
| 3. Healthy doesn't cost more | H4 | A/B costs **more** than D/E per 100 g in the same aisle | **Rejected.** A/B costs 0.83×. No aisle shows a significant health premium. |
| 4. What shoppers pay for | H5 | Less-processed food costs more | Per package, NOVA 1–3 costs +11% (95% CI 9–14%) |
| | H6 | Organic costs more | +18% per package (95% CI 15–22%). Ranges from +4% (sauces) to +54% (fats). |
| 5. Shop the store | H7 | Store brands are cheaper with no worse Nutri-Score | **Supported.** 0.58× per 100 g, +10.8 pp more A/B, −22.8 pp ultra-processed |
| | H8 | Ultra-processed products are promoted more, within store and aisle | **Weak.** Raw gaps are large (64% vs 39% at Nob Hill), but only +2.4 pp within store × aisle |

### Figures

| # | Figure |
|---|---|
| 1 | Nutri-Score mix by food group |
| 2 | % ultra-processed vs % Nutri-Score A/B by aisle |
| 3 | Nutri-Score mix by Green-Score impact level |
| 4 | Healthy vs low-impact quadrant by food group |
| 5 | Within-group correlation of Nutri-Score and Green-Score |
| 6 | Health gain from choosing the low-impact product |
| 7 | Food-group representation in the priced sample |
| 8 | Median price per 100 g by food group |
| 9 | A/B vs D/E price ratio within food group |
| 10 | Processing price premium |
| 11 | Nutrient-linked price differences |
| 12 | Organic premium by food group |
| 13 | Store-brand vs national-brand price ratio |
| 14 | Same barcode, different store |
| 15 | Promotion rate by processing level and store |

---

## The smart-shopper playbook

1. **Compare within the aisle.** The aisle decides most of a product's health score.
2. **Read the price per 100 g, not the package price.**
3. **Swap down a grade, not up in price.** Products graded A/B cost about 17% less per 100 g than D/E products in the same aisle.
4. **Choose the greener option.** It is usually healthier and about 7% cheaper.
5. **Try the store brand first.** It costs about 42% less per 100 g, partly because packs are bigger.
6. **Know your store.** Smart & Final and FoodMaxx are about 11% below average for the same barcode.
7. **Don't follow sale tags into the snack aisle.** Promotions follow the aisle, not the product's health.

### Next step: SmartCart swap-app prototype

The final section of the analysis notebook prototypes the logic for a gamified app. A shopper scans a product and sees its price per 100 g compared with the aisle median. The app suggests up to three swaps from the same aisle that are healthier and cheaper and share at least 50% of their category tags (Jaccard similarity). It also shows the cheapest store seen for that barcode, and awards points for each swap. Photographing price tags would add new prices to Open Prices, which helps fix the project's biggest data limitation.

---

## Limitations

- **Local, concentrated price data.** 85% of prices are from California and 93% from one contributor.
- **Small priced sample.** Only 5,752 foods (0.6% of the catalog) have both a price per 100 g and a food group. Package size is known for only 43% of priced products.
- **Coarse matching.** Food groups still mix different products (e.g. gallons of milk and yogurt cups), so part of each gap may come from product mix.
- **Approximate flags.** Store brands are identified with a brand-name regex. Organic status in Figure 12 comes from the word "organic" in the product name.
- **Not missing at random.** Popular products are more likely to have scores and prices.
- **Associations, not causes.**
- Drinks are excluded from per-100 g price comparisons. Figures 10–12 use **package** price indices, not price per 100 g.

---

## Team

| Member | Contribution |
|---|---|
| Aashna | Theme 1: Know your aisle |
| Gauri | Theme 2: The greener choice |
| Peter | Theme 3: Healthy doesn't cost more |
| Dhruv | Theme 4: What shoppers pay for |
| Alex | Theme 5: Shop the store |

## GenAI disclosure

We used Claude, ChatGPT, Codex, and Gemini as coding, debugging, and writing assistants. The full disclosure, including how we checked the AI output, is at the end of the analysis notebook. The team made all decisions about research questions, methods, and conclusions.

## References

- Open Food Facts. *Product database*. Hugging Face. ODbL 1.0.
- Open Food Facts. *Open Prices*. Hugging Face. ODbL 1.0.
- Open Food Facts. *Food-group taxonomy*. [github.com/openfoodfacts/openfoodfacts-server](https://github.com/openfoodfacts/openfoodfacts-server/tree/main/taxonomies)
- Santé publique France. *Nutri-Score*. [santepubliquefrance.fr/en/nutri-score](https://www.santepubliquefrance.fr/en/nutri-score)
- Monteiro, C. A., et al. (2019). Ultra-processed foods: what they are and how to identify them. *Public Health Nutrition*, 22(5), 936–941.
- Benjamini, Y., & Hochberg, Y. (1995). Controlling the false discovery rate. *Journal of the Royal Statistical Society B*, 57(1), 289–300.

## License

Data: Open Food Facts and Open Prices, [Open Database License (ODbL) 1.0](https://opendatacommons.org/licenses/odbl/1-0/). Any redistributed derived data (`input_data/`) must also be shared under ODbL.
