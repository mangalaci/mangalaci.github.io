---
layout: post
title: Demand Forecast of Retail Products Using ARIMA Method
image: "/posts/ab-testing-title-img.png"
tags: [Demand Forecast, ARIMA, Python]
---

In this project I evaluated a campaign perdormance among bank client who received reminders using control groups. The applied techniques comprised Chi-Square Test For Independence and 3-way ANOVA.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Context](#overview-context)
    - [Actions](#overview-actions)
    - [Results & Discussion](#overview-results)
- [01. Concept Overview](#concept-overview)
- [02. Data Overview & Preparation](#data-overview)
- [03. Applying Chi-Square Test For Independence](#chi-square-application)
- [04. Analysing The Results](#chi-square-results)
- [05. Discussion](#discussion)

___

# Project Overview  <a name="overview-main"></a>

### Context <a name="overview-context"></a>

In 2023, my client, a major Hungarian bank, ran a campaign to prevent their clients - who took a personal loan - from going into red. The goal of the campaign was 2-folded: on one hand it meant to collect the due installments on time - and thus saving some client to default, on the other hand enhance customer satisfaction.
Customers were put randomly into two groups - the first group received a reminder a couple of day before their due date, the second group did not receive any reminder.

The bank wanted to understand if there was a significant difference in payback rate between the reminder and the non-reminder groups.  This would allow them to make more informed decisions in the future, with the overall aim of optimising campaign ROI!

<br>
<br>
### Actions <a name="overview-actions"></a>

For this test, as it is focused on comparing the *rates* of two groups - we applied the Chi-Square Test For Independence.  Full details of this test can be found in the dedicated section below.

**Note:** Another option when comparing "rates" is a test known as the *Z-Test For Proportions*.  While, we could absolutely use this test here, we have chosen the Chi-Square Test For Independence because:

* The resulting test statistic for both tests will be the same
* The Chi-Square Test can be represented using 2x2 tables of data - meaning it can be easier to explain to stakeholders
* The Chi-Square Test can extend out to more than 2 groups - meaning the client can have one consistent approach to measuring signficance


We set out our hypotheses and Acceptance Criteria for the test, as follows:

**Null Hypothesis:** There is no relationship in due payment rate between those who received a reminder  and those who  did not. They are independent.
**Alternate Hypothesis:** There is a relationship between the two groups. They are not independent.
**Acceptance Criteria:** 0.05

As a requirement of the Chi-Square Test For Independence, we aggregated this data down to a 2x2 matrix for *reminder_flag* by *payment_lateness_type* and fed this into the algorithm (using the *scipy* library) to calculate the Chi-Square Statistic, p-value, Degrees of Freedom, and expected values

<br>
<br>

### Results & Discussion <a name="overview-results"></a>

Based upon our observed values, we can give this all some context with the sign-up rate of each group.  We get:

* Mailer 1 (Low Cost): **32.8%** signup rate
* Mailer 2 (High Cost): **37.8%** signup rate

However, the Chi-Square Test gives us the following statistics:

* Chi-Square Statistic: **1.94**
* p-value: **0.16**

The Critical Value for our specified Acceptance Criteria of 0.05 is **3.84**

Based upon these statistics, we retain the null hypothesis, and conclude that there is no relationship between mailer type and signup rate.

In other words - while we saw that the higher cost Mailer 2 had a higher signup rate (37.8%) than the lower cost Mailer 1 (32.8%) it appears that this difference is not significant, at least at our Acceptance Criteria of 0.05.

Without running this Hypothesis Test, the client may have concluded that they should always look to go with higher cost mailers - and from what we've seen in this test, that may not be a great decision.  It would result in them spending more, but not *necessarily* gaining any extra revenue as a result

Our results here also do not say that there *definitely isn't a difference between the two mailers* - we are only advising that we should not make any rigid conclusions *at this point*.  

Running more A/B Tests like this, gathering more data, and then re-running this test may provide us, and the client more insight!

<br>
<br>

___

# Concept Overview  <a name="concept-overview"></a>

<br>
#### A/B Testing

An A/B Test can be described as a randomised experiment containing two groups, A & B, that receive different experiences. Within an A/B Test, we look to understand and measure the response of each group - and the information from this helps drive future business decisions.

Application of A/B testing can range from testing different online ad strategies, different email subject lines when contacting customers, or testing the effect of mailing customers a coupon, vs a control group.  Companies like Amazon are running these tests in an almost never-ending cycle, testing new website features on randomised groups of customers...all with the aim of finding what works best so they can stay ahead of their competition.  Reportedly, Netflix will even test different images for the same movie or show, to different segments of their customer base to see if certain images pull more viewers in.

<br>
#### Hypothesis Testing

A Hypothesis Test is used to assess the plausibility, or likelihood of an assumed viewpoint based on sample data - in other words, a it helps us assess whether a certain view we have about some data is likely to be true or not.

There are many different scenarios we can run Hypothesis Tests on, and they all have slightly different techniques and formulas - however they all have some shared, fundamental steps & logic that underpin how they work.

<br>
**The Null Hypothesis**

In any Hypothesis Test, we start with the Null Hypothesis. The Null Hypothesis is where we state our initial viewpoint, and in statistics, and specifically Hypothesis Testing, our initial viewpoint is always that the result is purely by chance or that there is no relationship or association between two outcomes or groups

<br>
**The Alternate Hypothesis**

The aim of the Hypothesis Test is to look for evidence to support or reject the Null Hypothesis.  If we reject the Null Hypothesis, that would mean we’d be supporting the Alternate Hypothesis.  The Alternate Hypothesis is essentially the opposite viewpoint to the Null Hypothesis - that the result is *not* by chance, or that there *is* a relationship between two outcomes or groups

<br>
**The Acceptance Criteria**

In a Hypothesis Test, before we collect any data or run any numbers - we specify an Acceptance Criteria.  This is a p-value threshold at which we’ll decide to reject or support the null hypothesis.  It is essentially a line we draw in the sand saying "if I was to run this test many many times, what proportion of those times would I want to see different results come out, in order to feel comfortable, or confident that my results are not just some unusual occurrence"

Conventionally, we set our Acceptance Criteria to 0.05 - but this does not have to be the case.  If we need to be more confident that something did not occur through chance alone, we could lower this value down to something much smaller, meaning that we only come to the conclusion that the outcome was special or rare if it’s extremely rare.

So to summarise, in a Hypothesis Test, we test the Null Hypothesis using a p-value and then decide it’s fate based on the Acceptance Criteria.

<br>
**Types Of Hypothesis Test**

There are many different types of Hypothesis Tests, each of which is appropriate for use in differing scenarios - depending on a) the type of data that you’re looking to test and b) the question that you’re asking of that data.

In the case of our task here, where we are looking to understand the difference in sign-up *rate* between two groups - we will utilise the Chi-Square Test For Independence.

<br>
#### Chi-Square Test For Independence

The Chi-Square Test For Independence is a type of Hypothesis Test that assumes observed frequencies for categorical variables will match the expected frequencies.

The *assumption* is the Null Hypothesis, which as discussed above is always the viewpoint that the two groups will be equal.  With the Chi-Square Test For Independence we look to calculate a statistic which, based on the specified Acceptance Criteria will mean we either reject or support this initial assumption.

The *observed frequencies* are the true values that we’ve seen.

The *expected frequencies* are essentially what we would *expect* to see based on all of the data.

**Note:** Another option when comparing "rates" is a test known as the *Z-Test For Proportions*.  While, we could absolutely use this test here, we have chosen the Chi-Square Test For Independence because:

* The resulting test statistic for both tests will be the same
* The Chi-Square Test can be represented using 2x2 tables of data - meaning it can be easier to explain to stakeholders
* The Chi-Square Test can extend out to more than 2 groups - meaning the business can have one consistent approach to measuring signficance

___

<br>
# Data Overview & Preparation  <a name="data-overview"></a>

In this project, the contingency tables for ... ... were provided by data management team, thus importing data and creating contóingency tables were not needed since dara was given in a matrix format.


<br>
A sample of this data (the first 10 rows) can be seen below:
<br>
<br>

| **customer_id** | **campaign_name** | **mailer_type** | **signup_flag** |
|---|---|---|---|
| 74 | delivery_club | Mailer1 | 1 |
| 524 | delivery_club | Mailer1 | 1 |
| 607 | delivery_club | Mailer2 | 1 |
| 343 | delivery_club | Mailer1 | 0 |
| 322 | delivery_club | Mailer2 | 1 |
| 115 | delivery_club | Mailer2 | 0 |
| 1 | delivery_club | Mailer2 | 1 |
| 120 | delivery_club | Mailer1 | 1 |
| 52 | delivery_club | Mailer1 | 1 |
| 405 | delivery_club | Mailer1 | 0 |
| 435 | delivery_club | Mailer2 | 0 |

<br>
In the DataFrame we have:

* customer_id
* campaign name
* mailer_type (either Mailer1 or Mailer2)
* signup_flag (either 1 or 0)

___

<br>
# Applying Chi-Square Test For Independence <a name="chi-square-application"></a>

<br>
#### State Hypotheses & Acceptance Criteria For Test

The very first thing we need to do in any form of Hypothesis Test is state our Null Hypothesis, our Alternate Hypothesis, and the Acceptance Criteria (more details on these in the section above)

In the code below we code these in explcitly & clearly so we can utilise them later to explain the results.  We specify the common Acceptance Criteria value of 0.05.

```python

# specify hypotheses & acceptance criteria for test
null_hypothesis = "There is no relationship between mailer type and signup rate.  They are independent"
alternate_hypothesis = "There is a relationship between mailer type and signup rate.  They are not independent"
acceptance_criteria = 0.05

```

<br>
#### Calculate Observed Frequencies & Expected Frequencies

As mentioned in the section above, in a Chi-Square Test For Independence, the *observed frequencies* are the true values that we’ve seen, in other words the actual rates per group in the data itself.  The *expected frequencies* are what we would *expect* to see based on *all* of the data combined.

The below code:

* Summarises our dataset to a 2x2 matrix for *signup_flag* by *mailer_type*
* Based on this, calculates the:
    * Chi-Square Statistic
    * p-value
    * Degrees of Freedom
    * Expected Values
* Prints out the Chi-Square Statistic & p-value from the test
* Calculates the Critical Value based upon our Acceptance Criteria & the Degrees Of Freedom
* Prints out the Critical Value

```python

mport numpy as np
from scipy.stats import chi2_contingency

def perform_chi_square_test_and_print_results(contingency_table, table_name):
    """
    Perform chi-square test of independence for a given contingency table and print the results.
   
    Parameters:
    - contingency_table: 2D array, contingency table representing the counts or frequencies.
    - table_name: str, name of the contingency table for printing results.
   
    Returns:
    - chi2_stat: float, chi-square test statistic.
    - p_value: float, p-value of the test.
    - dof: int, degrees of freedom.
    - expected: 2D array, expected frequencies under the null hypothesis.
    """
    # Perform chi-square test of independence without Yates' continuity correction
    chi2_stat, p_value, dof, expected = chi2_contingency(contingency_table, correction=False)
   
    # Print results
    print(f"Results for {table_name}:")
    print("Chi-square statistic:", chi2_stat)
    print("P-value:", p_value)
    print()
   
    return chi2_stat, p_value, dof, expected

# Contingency tables for products
baby_craving = np.array([[508, 112],
                         [62, 16]])

personal_loan = np.array([[8909, 2163],
                           [2763, 706]])

mortgage_loan = np.array([[4459, 1136],
                           [102, 38]])


commodity_credit = np.array([[254, 47],
                           [221, 57]])


# Perform chi-square test of independence and print results for baby craving
perform_chi_square_test_and_print_results(baby_craving, "Baby Craving")

# Perform chi-square test of independence and print results for personal loan
perform_chi_square_test_and_print_results(personal_loan, "Personal Loan")

# Perform chi-square test of independence and print results for mortgage loan
perform_chi_square_test_and_print_results(mortgage_loan, "Mortgage Loan")

# Perform chi-square test of independence and print results for commodity credit
perform_chi_square_test_and_print_results(commodity_credit, "Commodity Credit")

```
<br>
Based upon our observed values, we can give this all some context with the sign-up rate of each group.  We get:

* Mailer 1 (Low Cost): **32.8%** signup rate
* Mailer 2 (High Cost): **37.8%** signup rate

From this, we can see that the higher cost mailer does lead to a higher signup rate.  The results from our Chi-Square Test will provide us more information about how confident we can be that this difference is robust, or if it might have occured by chance.

We have a Chi-Square Statistic of **1.94** and a p-value of **0.16**.  The critical value for our specified Acceptance Criteria of 0.05 is **3.84**

**Note** When applying the Chi-Square Test above, we use the parameter *correction = False* which means we are applying what is known as the *Yate's Correction* which is applied when your Degrees of Freedom is equal to one.  This correction helps to prevent overestimation of statistical signficance in this case.

___

<br>
# Analysing The Results <a name="chi-square-results"></a>

At this point we have everything we need to understand the results of our Chi-Square test - and just from the results above we can see that, since our resulting p-value of **0.16** is *greater* than our Acceptance Criteria of 0.05 then we will _retain_ the Null Hypothesis and conclude that there is no significant difference between the signup rates of Mailer 1 and Mailer 2.

We can make the same conclusion based upon our resulting Chi-Square statistic of **1.94** being _lower_ than our Critical Value of **3.84**

To make this script more dynamic, we can create code to automatically interpret the results and explain the outcome to us...

```python

# print the results (based upon p-value)
if p_value <= acceptance_criteria:
    print(f"As our p-value of {p_value} is lower than our acceptance_criteria of {acceptance_criteria} - we reject the null hypothesis, and conclude that: {alternate_hypothesis}")
else:
    print(f"As our p-value of {p_value} is higher than our acceptance_criteria of {acceptance_criteria} - we retain the null hypothesis, and conclude that: {null_hypothesis}")

>> As our p-value of 0.16351 is higher than our acceptance_criteria of 0.05 - we retain the null hypothesis, and conclude that: There is no relationship between mailer type and signup rate.  They are independent


# print the results (based upon p-value)
if chi2_statistic >= critical_value:
    print(f"As our chi-square statistic of {chi2_statistic} is higher than our critical value of {critical_value} - we reject the null hypothesis, and conclude that: {alternate_hypothesis}")
else:
    print(f"As our chi-square statistic of {chi2_statistic} is lower than our critical value of {critical_value} - we retain the null hypothesis, and conclude that: {null_hypothesis}")
    
>> As our chi-square statistic of 1.9414 is lower than our critical value of 3.841458820694124 - we retain the null hypothesis, and conclude that: There is no relationship between mailer type and signup rate.  They are independent

```
<br>
As we can see from the outputs of these print statements, we do indeed retain the null hypothesis.  We could not find enough evidence that the signup rates for Mailer 1 and Mailer 2 were different - and thus conclude that there was no significant difference.

___

<br>
# Discussion <a name="discussion"></a>

While we saw that the higher cost Mailer 2 had a higher signup rate (37.8%) than the lower cost Mailer 1 (32.8%) it appears that this difference is not significant, at least at our Acceptance Criteria of 0.05.

Without running this Hypothesis Test, the client may have concluded that they should always look to go with higher cost mailers - and from what we've seen in this test, that may not be a great decision.  It would result in them spending more, but not *necessarily* gaining any extra revenue as a result

Our results here also do not say that there *definitely isn't a difference between the two mailers* - we are only advising that we should not make any rigid conclusions *at this point*.  

Running more A/B Tests like this, gathering more data, and then re-running this test may provide us, and the client more insight!

