# License Calculation – Synthetic and RUM

Synthetic monitoring and RUM are billed **pay as you go**: you pay for every HTTP request or browser action a monitor runs, and for every real user session. The UI doesn't show the calculation, but you can see your usage in **Account Management** (bottom left) **→ Subscription**.

## 1. Rate card

| # | Capability | Billed per | Price | Price per unit |
|---|------------|------------|-------|----------------|
| 1 | HTTP monitor | Synthetic request | 1 USD per 1,000 requests | 0.001 USD |
| 2 | Browser monitor | Synthetic action | 4.5 USD per 1,000 actions | 0.0045 USD |
| 3 | Real User Monitoring (RUM) | Session | 2.25 USD per 1,000 sessions | 0.00225 USD |
| 4 | RUM with Session Replay | Session | 4.5 USD per 1,000 sessions | 0.0045 USD |

Source: [Dynatrace pricing](https://www.dynatrace.com/pricing/)

Cost formula:

```text
cost = URLs × locations × executions per hour × hours × price per request (or action)
```

## 2. HTTP monitor cost

Example (`UC_01`): 1 URL, 4 locations, runs every 5 minutes (12 times an hour), at **0.001 USD per request**.

| Period | Requests | Cost (USD) |
|--------|----------|------------|
| 1 hour | 1 × 4 × 12 = **48** | 48 × 0.001 = **0.048** |
| 1 day | 48 × 24 = **1,152** | **1.15** |
| 1 year | 1,152 × 365 = **420,480** | **420.48** |

> Cost grows linearly with each factor. Running every 15 minutes instead of 5 cuts the cost to a third; running from 2 locations instead of 4 halves it.

## 3. Browser monitor cost

The same example (`UC_01`) as a single-URL browser monitor (1 page load = 1 action per run), at **0.0045 USD per action**.

| Period | Actions | Cost (USD) |
|--------|---------|------------|
| 1 hour | 1 × 4 × 12 = **48** | 48 × 0.0045 = **0.216** |
| 1 day | 48 × 24 = **1,152** | **5.18** |
| 1 year | 1,152 × 365 = **420,480** | **1,892.16** |

> A clickpath monitor is billed for **every action** in the recorded journey. A 5-step clickpath costs 5× the single-URL example: about **9,460.80 USD per year**.

## 4. Network monitoring

Doesnot cost anything as pinged from internal network 