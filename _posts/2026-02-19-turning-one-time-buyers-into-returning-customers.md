---
layout: post
title: Turning One-Time Buyers Into Returning Customers Using Product Journey Analysis
image: "/posts/returning-customers-title.png"
tags: [Python, Pandas, CRM, Customer Analytics, E-commerce, Repeat Customers]
---

In optical e-commerce, roughly 65% of buyers never come back. This project identifies what separates one-time buyers from repeat customers by analyzing 2.1 million transactions across 277,000 users. By tracing each customer's full product journey — from their very first basket to subsequent purchases — I uncovered which products, sequences, and timing patterns predict who will return. The result: 12 data-driven CRM campaigns designed to convert one-time buyers into returning customers, with an estimated combined revenue impact of ~35 million.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Business Problem](#overview-problem)
    - [Approach](#overview-approach)
    - [Key Results](#overview-results)
- [01. What Does the First Purchase Tell Us?](#first-purchase)
    - [Basket Classification](#basket-classification)
    - [Repeat Rate by First Basket](#repeat-by-basket)
    - [Lens Parameters and Brand](#lens-brand)
- [02. What Happens When They Come Back?](#transitions)
    - [Purchase Transitions](#transition-matrix)
    - [Brand Loyalty](#brand-loyalty)
- [03. Golden Paths vs Dead-End Paths](#golden-paths)
    - [2-Step Paths](#two-step)
    - [3-Step Paths](#three-step)
- [04. Can Cross-Selling Turn One-Timers Into Repeaters?](#cross-sell)
- [05. When Do Customers Come Back?](#timing)
    - [Reorder Gaps by Product](#timing-gaps)
    - [Late Returners and Reminder Timing](#timing-reminders)
- [06. 12 CRM Campaigns to Convert One-Timers](#campaigns)
    - [Campaign Details](#campaign-details)
    - [Combined Impact Estimate](#impact-estimate)
- [07. Key Takeaways & Next Steps](#takeaways)

___

# Project Overview  <a name="overview-main"></a>

<br>
### Business Problem <a name="overview-problem"></a>

A European optical e-commerce retailer with 277,000 customers faces a common challenge: **most first-time buyers never return**. Out of all customers who make their first purchase, only 34.4% ever come back for a second one. The remaining 65.6% are lost after a single transaction.

The cost of this is significant. A repeat buyer generates a median lifetime revenue of 17,890, compared to 7,874 for a one-timer — an incremental value of **10,016 per converted customer**. Even converting a small fraction of one-timers into repeaters would have substantial revenue impact.

But blanket "come back" campaigns are inefficient. We need to understand **who** is likely to return, **what** product patterns predict return behavior, and **when** to intervene.

<br>
### Approach <a name="overview-approach"></a>

I built a Python analytics pipeline that traces the full **product journey** of every customer:

1. **Data Preparation**: 2.1 million transaction rows cleaned (outliers removed via p99.9 percentile), aggregated from line-item level to invoice-level purchase events, and each purchase classified into one of 10 basket types
2. **First Basket Analysis**: Which first purchases predict who will return?
3. **Transition Analysis**: When customers do return, what do they buy next?
4. **Path Analysis**: Which multi-step purchase sequences are "golden paths" (high future retention) vs "dead ends"?
5. **Cross-sell Analysis**: Does adding accessories to a purchase improve the likelihood of a next purchase?
6. **Timing Analysis**: When do customers typically reorder, and how does this vary by product?
7. **Campaign Generation**: Translate all findings into actionable CRM campaigns with estimated business impact

<br>
### Key Results <a name="overview-results"></a>

* **34.4%** overall repeat rate (65.6% of buyers are one-timers)
* **6.3x** range in repeat rate by first basket type (10.0% to 63.4%)
* **86%** brand loyalty among contact lens repeat buyers
* **12** evidence-based CRM campaigns generated
* **~35M** estimated combined revenue impact (at 15% campaign take-up)

___

# What Does the First Purchase Tell Us?  <a name="first-purchase"></a>

The first purchase is our strongest predictor of whether someone will return. If we can understand which first baskets predict return behavior, we can score every new customer on day one and trigger the right intervention immediately.

<br>
### Basket Classification <a name="basket-classification"></a>

To make purchase patterns analyzable, I classified each purchase into a basket type based on what product groups it contains. A customer who buys contact lenses plus cleaning solution has a fundamentally different profile than someone who buys sunglasses only:

```python
def classify_basket(pg_list):
    has_cl  = 'Contact lenses' in pg_list
    has_clt = 'Contact lenses - Trials' in pg_list
    has_sol = 'Contact lens cleaners' in pg_list
    has_eye = 'Eye drops' in pg_list
    has_frm = 'Frames' in pg_list or 'Lenses for spectacles' in pg_list
    has_sun = 'Sunglasses' in pg_list

    if has_clt and not has_cl:
        return 'CL Trials'
    if has_cl and has_sol and has_eye:
        return 'CL + Solution + EyeDrop'
    if has_cl and has_sol:
        return 'CL + Solution'
    if has_cl and has_eye:
        return 'CL + EyeDrop'
    if has_cl:
        return 'CL only'
    if has_sol and has_eye:
        return 'Solution + EyeDrop'
    if has_sol:
        return 'Solution only'
    if has_eye:
        return 'EyeDrop only'
    if has_frm:
        return 'Spectacles/Frames'
    if has_sun:
        return 'Sunglasses'
    return 'Other'
```

The hierarchy is intentional: contact lens combinations are checked first because they represent the highest-value, most predictable customer segment.

<br>
### Repeat Rate by First Basket <a name="repeat-by-basket"></a>

The results are striking. Across 230,392 first-time buyers, the repeat rate varies by **over 6x** depending on what the customer bought first:

| First Basket Type | n | % of users | Repeat Rate | Lift vs avg |
|---|---|---|---|---|
| CL + Solution + EyeDrop | 2,098 | 0.9% | 63.4% | +29.0pp |
| CL + Solution | 16,248 | 7.1% | 59.3% | +24.9pp |
| Solution + EyeDrop | 1,814 | 0.8% | 57.2% | +22.8pp |
| CL + EyeDrop | 2,041 | 0.9% | 56.7% | +22.3pp |
| CL only | 64,609 | 28.0% | 53.0% | +18.6pp |
| CL Trials | 1,906 | 0.8% | 51.9% | +17.5pp |
| Solution only | 19,781 | 8.6% | 42.9% | +8.5pp |
| EyeDrop only | 23,376 | 10.1% | 34.0% | -0.4pp |
| Other | 6,164 | 2.7% | 20.5% | -13.9pp |
| Spectacles/Frames | 40,479 | 17.6% | 19.8% | -14.6pp |
| Sunglasses | 51,876 | 22.5% | 10.0% | -24.5pp |

The pattern is clear: **consumable products drive repeat behavior**. Contact lenses need replacing — the product itself creates a reason to return. Sunglasses and frames are one-time purchases by nature.

The most powerful finding: customers who buy a **bundle** (CL + Solution + EyeDrop) in their first purchase have a 63.4% repeat rate, nearly double the average. This suggests that a more complete first basket indicates a committed customer — and also that nudging first-time buyers toward bundles could increase their return likelihood.

<br>
### Lens Parameters and Brand <a name="lens-brand"></a>

Within contact lens first-buyers (n=84,996), both the lens wear schedule and brand further predict return behavior:

**By wear schedule:**

| Wear Days | n | Repeat Rate |
|---|---|---|
| 90-day | 26,319 | 61.2% |
| 84-day | 5,093 | 56.7% |
| 180-day | 24,394 | 51.8% |
| 30-day | 13,140 | 48.2% |

90-day lenses have the highest repeat rate, likely because the 3-month cycle is long enough to create habit but short enough to need regular reordering.

**By brand (top and bottom):**

| Brand | n | Repeat Rate |
|---|---|---|
| VitaFlex | 25,327 | 67.2% |
| OptiClean | 4,084 | 66.3% |
| EyeCare Pure | 1,473 | 65.4% |
| AeroVue | 16,483 | 56.8% |
| AquaVue | 10,812 | 53.8% |
| DayLens | 6,879 | 46.7% |
| StyleLook | 1,125 | 33.9% |
| ColorLens | 6,219 | 21.3% |

VitaFlex buyers return at 67.2% — more than 3x the rate of ColorLens buyers (21.3%). ColorLens is a cosmetic/colored lens brand with many impulse buyers, while VitaFlex is a premium daily-wear brand whose users have an ongoing need.

**Takeaway:** The first basket — its composition, product parameters, and brand — is a powerful signal. We can score every new customer's likelihood to return from day one and tailor the follow-up accordingly.

___

# What Happens When They Come Back?  <a name="transitions"></a>

For the 34.4% of customers who do make a second purchase, the next question is: what do they buy, and does it predict whether they'll make a third? Understanding purchase transitions tells us what to recommend to returning customers and whether brand loyalty is a lever we can pull.

<br>
### Purchase Transitions <a name="transition-matrix"></a>

Among 86,273 users with both a first (P1) and second (P2) purchase, the dominant pattern is **same-category repurchase**:

| P1 Basket | P2 Basket | n | % |
|---|---|---|---|
| CL only | CL only | 27,465 | 31.8% |
| Spectacles/Frames | Spectacles/Frames | 7,767 | 9.0% |
| EyeDrop only | EyeDrop only | 7,141 | 8.3% |
| Sunglasses | Sunglasses | 6,419 | 7.4% |
| Solution only | Solution only | 6,327 | 7.3% |
| CL + Solution | CL + Solution | 4,088 | 4.7% |
| CL + Solution | CL only | 3,795 | 4.4% |
| CL only | CL + Solution | 3,328 | 3.9% |

Most customers repeat the same category. But some transitions are more interesting:
- **3,328 users** went from "CL only" to "CL + Solution" — they added an accessory on return
- **3,795 users** went from "CL + Solution" to "CL only" — they dropped the accessory

Which transitions predict a **third purchase** (P3)?

| P1 → P2 | n | P3 Rate |
|---|---|---|
| CL + Solution → CL + Solution | 4,088 | 79.8% |
| CL only → CL + Solution + EyeDrop | 373 | 79.4% |
| Sunglasses → Sunglasses | 6,419 | 26.6% |
| Spectacles/Frames → Spectacles/Frames | 7,767 | 38.5% |

CL + Solution → CL + Solution is the strongest high-volume path: nearly 80% of these users make a third purchase. In contrast, Sunglasses → Sunglasses has only 26.6% — two sunglasses purchases does not predict a third.

<br>
### Brand Loyalty <a name="brand-loyalty"></a>

Among CL buyers who purchased CL at both P1 and P2:

| Loyalty | n | % | P3 Rate |
|---|---|---|---|
| Loyal (same brand) | 37,716 | 86.0% | 74.6% |
| Switched brand | 6,142 | 14.0% | 68.6% |

**86% of returning CL customers buy the same brand.** This is a powerful lever: if we know what brand they bought, we can pre-fill their reorder with the exact same product. Brand loyalists also have 6pp higher P3 rates — they're on a more stable path.

**Takeaway:** Returning customers are creatures of habit. Pre-filling reorders with their previous brand and product makes the path of least resistance lead to another purchase.

___

# Golden Paths vs Dead-End Paths  <a name="golden-paths"></a>

Individual purchases tell part of the story, but the full **sequence** is more predictive. Some 2-step and 3-step purchase paths almost guarantee another purchase, while others are dead ends. If we can detect which path a customer is on after their second purchase, we can reinforce golden paths or intervene on dead ends.

<br>
### 2-Step Paths <a name="two-step"></a>

Among 86,273 users with 2+ purchases, the baseline P3 rate is 62.0%. Some paths far exceed this:

**Top golden paths (highest P3 rate):**

```
[4,088] P3=79.8% (+17.8pp) | CL + Solution -> CL + Solution
[  373] P3=79.4% (+17.3pp) | CL only -> CL + Solution + EyeDrop
[  243] P3=79.0% (+17.0pp) | CL + Solution + EyeDrop -> CL + Solution + EyeDrop
```

**Top dead-end paths (lowest P3 rate):**

```
[7,767] P3=38.5% (-23.5pp) | Spectacles/Frames -> Spectacles/Frames
[  785] P3=35.9% (-26.1pp) | Spectacles/Frames -> Sunglasses
[6,419] P3=26.6% (-35.4pp) | Sunglasses -> Sunglasses
```

The contrast is dramatic: CL + Solution → CL + Solution has nearly **3x** the P3 rate of Sunglasses → Sunglasses.

<br>
### 3-Step Paths <a name="three-step"></a>

For users with 3+ purchases (n=53,495), the baseline P4 rate is 73.0%. The best 3-step paths reach near-certain future purchases:

```
[42] P4=95.2% (+22.3pp) | CL+Sol+Eye -> CL+Sol+Eye -> CL+Sol
[80] P4=90.0% (+17.0pp) | CL+Sol+Eye -> CL+Sol+Eye -> CL+Sol+Eye
```

While the worst 3-step paths show persistent churn:

```
[1,479] P4=31.3% (-41.7pp) | Sunglasses -> Sunglasses -> Sunglasses
```

**Takeaway:** Purchase paths are more predictive than individual purchases. A customer who has bought CL + Solution twice is almost certainly going to buy again. A customer who has bought sunglasses twice is probably done. CRM systems should track paths, not just purchase counts, to decide when and how to intervene.

___

# Can Cross-Selling Turn One-Timers Into Repeaters?  <a name="cross-sell"></a>

If a CL-only buyer adds solution or eye drops at their second purchase, does that make them more likely to come back a third time? This directly tests whether cross-sell campaigns can drive repeat behavior — or whether the accessory purchase is just a side effect of being a committed customer.

Starting population: **64,609 CL-only first buyers**. Of these, only 53.4% made a second purchase (34,480 users). Among those who returned:

| Group | n | P3 Rate |
|---|---|---|
| Added solution at P2 | 4,860 | 74.4% |
| Added eye drops at P2 | 1,301 | 72.8% |
| Added any accessory at P2 | 5,692 | 73.7% |
| No accessory at P2 | 28,788 | 71.8% |

Adding accessories at P2 lifts the P3 rate by **+1.9 percentage points**. This seems small, but over the 28,788 users who didn't add accessories, even a 15% campaign take-up would yield ~82 additional P3 buyers.

The effect is modest, which suggests that cross-selling is **partially** a signal (committed customers are more likely to buy accessories AND return) rather than purely causal. But even a partially causal effect is worth pursuing at this scale — the intervention cost (an email recommendation) is near zero.

**Takeaway:** Cross-sell has a measurable, if modest, effect on repeat behavior. "Customers who buy [brand] also use [solution]" recommendations in reorder emails are low-cost, high-upside interventions.

___

# When Do Customers Come Back?  <a name="timing"></a>

Knowing WHEN customers typically reorder allows us to send reminders at exactly the right moment — before they forget, run out, or switch to a competitor. But as we'll see, most customers reorder much later than their product lifecycle would suggest.

<br>
### Reorder Gaps by Product <a name="timing-gaps"></a>

For CL first-buyers who did return, the gap between first and second purchase varies dramatically by wear schedule:

| Wear Days | n | Median Gap | Expected Window | In Window |
|---|---|---|---|---|
| 30-day | 7,044 | 94 days | 20-45 days | 17.3% |
| 90-day | 15,980 | 120 days | 70-110 days | 24.1% |
| 180-day | 12,716 | 190 days | 150-210 days | 17.3% |

The mismatch is remarkable: **30-day lens users take a median of 94 days to reorder** — more than 3x their lens supply. Only 17.3% reorder within the expected 20-45 day window. This means most customers are either:
- Stretching their lenses beyond recommended use
- Buying from competitors in between
- Simply forgetting to reorder

The gap distribution for 30-day lenses tells the story:

| Gap Bucket | n | % |
|---|---|---|
| <= 30 days | 1,295 | 18.4% |
| 31-60 days | 1,221 | 17.3% |
| 61-90 days | 789 | 11.2% |
| 91-180 days | 1,297 | 18.4% |
| 181-365 days | 1,121 | 15.9% |
| > 365 days | 1,182 | 16.8% |

The distribution is almost flat — there's no strong natural reorder cadence. This means **reminder campaigns can significantly shape behavior**.

<br>
### Late Returners and Reminder Timing <a name="timing-reminders"></a>

What fraction of customers return "late" (after 2x their expected window)?

| Wear Days | Late Threshold | % Late |
|---|---|---|
| 30-day | > 90 days | 51.1% |
| 90-day | > 220 days | 26.4% |
| 180-day | > 420 days | 19.5% |

**Over half of 30-day lens users are "late returners."** These are the prime targets for reorder reminders.

Based on the P25 of the gap distribution (the point at which 25% of returners have already come back), optimal reminder timing would be:

| Wear Days | Send Reminder At | Median Return |
|---|---|---|
| 30-day | Day 37 | Day 94 |
| 90-day | Day 77 | Day 120 |
| 180-day | Day 101 | Day 190 |

**Takeaway:** Most customers reorder far later than their product lifecycle suggests. Automated reminders sent at the P25 mark can catch the early majority, with follow-ups for those who don't respond. The 30-day lens segment is the biggest opportunity: 51% return late, and a well-timed reminder could pull many of them forward.

___

# 12 CRM Campaigns to Convert One-Timers  <a name="campaigns"></a>

The final step translates every analytical finding into an actionable CRM campaign with a specific target audience, action, and estimated business impact (assuming a conservative 15% campaign take-up rate).

<br>
### Campaign Details <a name="campaign-details"></a>

**1. Cross-sell solution at first CL purchase**
- Target: CL-only first buyers (n=64,609)
- Action: Show solution bundle offer during checkout or post-purchase email
- Impact: CL+Solution repeat rate is +6.3pp higher than CL-only. At 15% take-up: ~608 additional repeaters
- How: Trigger when cart contains CL but no solution → show bundle discount

**2. Push accessories in second purchase**
- Target: CL-only P1 buyers who made P2 without accessory (n=28,788)
- Action: Include solution/eye drop recommendations in reorder reminder
- Impact: +1.9pp P3 lift. At 15% take-up: ~82 additional P3 buyers
- How: Personalized email: "Others who use [brand] also use [solution]"

**3-5. Reorder reminders by wear schedule**
- 30-day lenses (n=7,044): remind at day 20, follow-up at day 45
- 90-day lenses (n=15,980): remind at day 70, follow-up at day 110
- 180-day lenses (n=12,716): remind at day 150, follow-up at day 210

**6. Leverage brand loyalty**
- Target: CL repeat buyers (86% stay with same brand, n=86,273)
- Action: Pre-fill reorder with same brand/product; offer loyalty discount
- How: "Reorder" button with pre-filled brand + parameters in email/app

**7. Win-back dead-end path users**
- Target: Users on low-retention paths like Sunglasses → Sunglasses (n=6,827)
- Action: Targeted win-back with discount or product change suggestion
- Impact: P3 rate as low as 28% for worst paths vs 62% baseline

**8. Trial-to-subscription conversion**
- Target: CL Trials first buyers (n=1,906)
- Action: Follow-up email with full product recommendation after trial period
- How: Trigger 7-14 days after trial purchase: "Ready for the full product?"

**9. Nurture Spectacles/Frames buyers into CL category**
- Target: Spectacles/Frames first buyers (n=40,479)
- Action: Cross-category campaign: introduce CL as complement to glasses
- Impact: At 15% take-up: ~888 potential additional repeaters

**10. Convert eye-drop-only buyers**
- Target: EyeDrop-only first buyers (n=23,376)
- Action: Educational content + CL trial offer

**11. Engage Sunglasses-only buyers**
- Target: Sunglasses first buyers (n=51,876, repeat rate only 10.0%)
- Action: Cross-category campaign introducing CL or prescription lenses
- Impact: At 15% take-up: ~1,902 potential additional repeaters

**12. Re-engage late returners**
- Target: CL users past 2x their expected reorder window (n=35,740)
- Action: "We miss you" email with discount code
- How: Trigger at 2x expected window

<br>
### Combined Impact Estimate <a name="impact-estimate"></a>

| Campaign | Target | Extra Repeaters | Est. Revenue |
|---|---|---|---|
| Cross-sell solution at first CL purchase | 64,609 | 608 | 6,089,570 |
| Push accessories at P2 | 28,788 | 82 | 821,291 |
| Nurture Spectacles/Frames into CL | 40,479 | 888 | 8,893,977 |
| Engage Sunglasses-only buyers | 51,876 | 1,902 | 19,049,937 |
| **TOTAL** | | **3,480** | **34,854,775** |

Incremental revenue per retained customer: 10,016 (median lifetime revenue of repeat buyer minus one-timer).

**Important caveats:**
- These are upper-bound estimates — campaigns may overlap (same user targeted by multiple campaigns)
- The observed lift is **correlational, not causal** — customers who buy bundles may be inherently more loyal, not made more loyal by the bundle
- **A/B testing** is recommended to validate each campaign before full rollout

___

# Key Takeaways & Next Steps  <a name="takeaways"></a>

### Top 3 Insights

1. **The first basket predicts return probability.** A customer who buys CL + Solution + EyeDrop has a 63.4% chance of returning vs 10.0% for sunglasses-only. We can score customers from day one and tailor interventions immediately.

2. **Purchase paths are more predictive than individual purchases.** CL + Solution → CL + Solution yields 79.8% P3 rate, while Sunglasses → Sunglasses yields 26.6%. Tracking sequences, not just counts, enables smarter CRM triggers.

3. **Timing is misaligned.** Most customers reorder far later than their product lifecycle suggests (30-day lenses: median reorder at 94 days). Well-timed reminders can pull forward purchases that would otherwise be lost.

### Planned Enhancements

1. **Survival curves**: Kaplan-Meier style analysis showing the probability of reorder over time, handling right-censoring properly for recent cohorts
2. **Cohort retention matrix**: Monthly cohorts tracking what % of users are still active at month 1, 2, 3, ... — enabling comparison across acquisition periods
3. **Channel segmentation**: Breaking down all metrics by webshop/channel to identify if store vs. online vs. cross-channel behavior differs
4. **Multichannel analysis**: Testing the hypothesis that customers who buy from multiple channels are more loyal

This project demonstrates how **product journey analysis** — tracing multi-step purchase sequences rather than looking at isolated transactions — can uncover actionable patterns that traditional RFM or cohort analysis would miss. The key shift is from asking "who is likely to churn?" to "what purchase path is this customer on, and how can we steer it toward the golden path?"
