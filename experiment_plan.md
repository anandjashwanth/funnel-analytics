# Experiment plan: raising product-view to add-to-cart

## The problem

Of 77,020 sessions that viewed a product (1 Nov 2020 to 31 Jan 2021), only 15,173 added an item to the cart: 19.7%. Every later step converts far better, so this is the biggest leak in the funnel.

| Step | Sessions | Step conversion |
|---|---|---|
| 1 Viewed item | 77,020 | |
| 2 Added to cart | 15,173 | 19.7% |
| 3 Began checkout | 5,959 | 39.3% |
| 4 Payment info | 3,858 | 64.7% |
| 5 Purchased | 2,848 | 73.8% |

Device is not the cause: full-funnel conversion is 0.82% on mobile, 0.77% on tablet and 0.77% on desktop.

## Hypothesis

If the product page shows the delivery cost and expected delivery date next to the Add to cart button, the add-to-cart rate will rise, because shoppers will not need to leave the page to find out the full cost and timing.

This is a hypothesis to test, not a known cause. The data shows where people drop out, not why; user research or session recordings would help choose between this and other ideas (price, stock, photos).

## Design

| Part | Choice | Why |
|---|---|---|
| Unit of randomisation | User (`user_pseudo_id`), 50/50 | A user always sees the same version, even across visits |
| Who is included | Sessions that view a product page | Only they can be affected by the change |
| Primary metric | Add-to-cart rate: sessions that add to cart / sessions that view a product | Measures the step being changed |
| Guardrail metrics | Overall conversion (baseline 1.35%), revenue per session (baseline $1.01), checkout-to-purchase rate (baseline 73.8%) | More carts are only good if orders and revenue do not fall |
| Minimum detectable effect | 10% relative: 19.7% to 21.7% | Smallest lift worth building; a smaller target needs a test too long to run (below) |
| Significance level | 5% (two-sided) | Standard |
| Power | 80% | Standard |

## Sample size and duration

Calculated in R:

```r
power.prop.test(p1 = 0.197, p2 = 0.2167, sig.level = 0.05, power = 0.80)
```

| Target lift | Sessions needed per group | Total | Days at ~840 eligible sessions a day |
|---|---|---|---|
| 10% relative (19.7% to 21.7%) | about 6,635 | about 13,270 | about 16 days, so run 3 full weeks |
| 5% relative (19.7% to 20.7%) | about 26,070 | about 52,150 | about 62 days, too long |

Eligible traffic: 77,020 product-viewing sessions over 92 days is about 840 a day. Running whole weeks covers weekday and weekend behaviour.

Note: the sample is counted in sessions while randomisation is by user. Sessions from the same user are not fully independent, so the real test should use a slightly larger sample or user-level analysis.

## Decision rule (set before the test starts)

- **Ship** if add-to-cart rate rises with p < 0.05 and no guardrail falls significantly.
- **Do not ship** if any guardrail falls, even if the primary metric rises.
- **Inconclusive** if neither: keep the current page and try the next hypothesis.

## Risks and checks

- **Peeking:** do not stop early when results first look significant; run the full 3 weeks.
- **Sample ratio mismatch:** check the split is close to 50/50 before reading results; a large gap means the randomisation is broken.
- **Novelty effect:** a new page element can draw clicks just for being new; 3 weeks helps it wear off.
- **Seasonality:** this baseline includes the Christmas peak; re-check the baseline in a normal month before launch.
- **Funnel coverage:** the strict funnel captures 2,848 of the 4,848 orders (it misses 41.3%), because some purchases skip earlier events in this dataset. The test should measure add-to-cart directly from events, not from the strict funnel.
