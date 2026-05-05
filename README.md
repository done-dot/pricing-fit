# pricing-fit

A Claude Code skill that guides you from raw event data to a done.finance outcome-based pricing contract in 6 steps.

## The CAMP Framework

CAMP is a four-dimension readiness framework for outcome-based pricing. Each dimension is scored 1–5; the total determines your phase.

**Consistency:** Do all customers value the same outcomes? Or do outcomes need to be customized, leading to a proliferation of bespoke contracts?

**Attribution:** Can you convince customers to give your product credit for delivering the outcome? Or do they believe they drove the outcome, with a small assist from you?

**Measurability:** Can you measure and report on the outcomes in real-time? Or do you require customer reporting, A/B testing and/or a proof of concept?

**Predictability:** Can you predict the outcomes your product will achieve with some level of accuracy? Or do outcomes vary significantly from customer to customer?

### Readiness phases

| Total score | Phase | What it means |
|-------------|-------|---------------|
| 4–8 | **Measure** | Instrument your product and run outcomes in shadow mode before charging |
| 9–15 | **Attribute** | Analyse confirmed rates and run a pilot with your most-engaged customers |
| 16–20 | **Price** | You're ready. Set the price and start signing outcome-based contracts |

## Usage

Run `/pricing-fit` in Claude Code to start the guided setup, or install this package and invoke the `pricing-fit` skill directly.

## Installation

```bash
/plugin marketplace add <your-github-username>/pricing-fit
/plugin install pricing-fit@<your-github-username>-pricing-fit
```
