---
layout: post
title: Assessing Campaign Performance in Reminding Bank Clients of Due Installment Payments
image: "/posts/ab-testing-title-img.png"
tags: [Chi-Square Test, Log-Linear Model, Python, AB Testing]
---

In this project I evaluated a campaign performance among bank clients who received reminders using control groups. The applied techniques comprised Chi-Square Test For Independence and Log-Linear Model.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Context](#overview-context)
    - [Actions](#overview-actions)
    - [Results & Discussion](#overview-results)
- [01. Statistical Modeling Consideration](#concept-overview)
- [02. Data Overview & Preparation](#data-overview)
- [03. Applying Models: Focused Analysis & Joint Effect with Treatment](#applying-models)
- [04. Analysing The Results](#chi-square-results)
- [05. Discussion](#discussion)

___

# Project Overview  <a name="overview-main"></a>

### Context <a name="overview-context"></a>

In 2023, my client, MBH Bank, the second largest Hungarian bank, ran a campaign to prevent their clients - who took a personal loan - from going into red. The goal of the campaign was 2-folded: on one hand it meant to collect the due installments on time - and thus saving some client to default, on the other hand enhance customer satisfaction.
Customers were put randomly into two groups - the first group received a reminder a couple of day before their due date, the second group did not receive any reminder.

The bank wanted to understand if there was a significant difference in payback rate between the reminder and the non-reminder groups.  This would allow them to make more informed decisions in the future, with the overall aim of optimising campaign ROI!

<br>
### Actions <a name="overview-actions"></a>

We focused on comparing the *rates* of two groups. However, our data on overall treatment-response and within products are represented in 2-dimensional tables of data, while when interaction of demographic caracteristics are taken into account we face 3-dimensional tables.

- we applied the Chi-Square Test For Independence.


* The Chi-Square Test can be represented using 2x2 tables of data - meaning it can be easier to explain to stakeholders
* The Chi-Square Test can extend out to more than 2 groups - meaning the client can have one consistent approach to measuring signficance


We set out our hypotheses and Acceptance Criteria for the test, as follows:

**Null Hypothesis:** There is no relationship in due payment rate between those who received a reminder and those who did not. They are independent.
**Alternate Hypothesis:** There is a relationship between the two groups. They are not independent.
**Acceptance Criteria:** 0.05

As a requirement of the Chi-Square Test For Independence, we aggregated this data down to a 2x2 matrix for *reminder_flag* by *payment_lateness_type* and fed this into the algorithm (using the *scipy* library) to calculate the Chi-Square Statistic, p-value, Degrees of Freedom, and expected values

<br>


### Results & Discussion <a name="overview-results"></a>

We examined the overall campaign efficiancy among all reminder receiving clients. On top of that, we took a closer look at the campaign effectiveness within products to enable us to spot high and low performing products. To fine-tune our campaings, we looked at how demographic attribues - age group and urbanization level (settlement type) - affect the late payment rate. This, however, involves a 3-way ANOVA analysis rather than simple Chi-Square Test.

Based upon our observed values, we can give this all some context with the late payment rate of each group.  We get:

##### All clients:

| **reminder received** | **reminder NOT received (control group)** |
|---|---|---|---|
| 19.6% | 20.7% |


<br>
##### Product-wise:

| **product** | **reminder received** | **reminder NOT received (control group)** |
|---|---|---|---|
| Baby Craving: | 10.9% | 12.5% |
| Personal Loan | 23.7% | 24.6% |
| Mortgage Loan | 2.2% | 3.2% |
| Commodity Credit | 46.5% | 54.8% |

<br>


However, the Chi-Square Test gives us the following statistics:

##### All clients:


| **Chi-Square Statistic** | **p-value** |
|---|---|---|
| 2.12 | 0.146 |


<br>
##### Product-wise:

| **product** | **Chi-Square Statistic** | **p-value** |
|---|---|---|---|
| Baby Craving | 0.277 | 0.598 |
| Personal Loan | 1.11 | 0.292 |
| Mortgage Loan | 3.924 | 0.048 |
| Commodity Credit | 2.344 | 0.126 |

The Critical Value for our specified Acceptance Criteria of 0.05 is **3.84**

<br>
Based upon these statistics, for all clients combined together we retain the null hypothesis, and conclude that reminder email has a NO effect on late payment rate. In other words - while we saw that the sending out a reminder had a lower late payment rate (19.6%) than for those who did not receive any reminder (20.7%) it appears that this difference is not significant, at least at our Acceptance Criteria of 0.05.

Without running this Hypothesis Test, the client may have concluded that they should always look to go with late payment reminder - and from what we've seen in this test, that may not be a great decision.


Product-wise, in case of:
* Mortgage Loan, we reject the null hypothesis, and conclude that the reminder email has a positive effect on late payment rate,

while in case of:
* Baby Craving
* Personal Loan
* Commodity Credit
we retain the null hypothesis, and conclude that reminder email has a NO effect on late payment rate.


From the results above we can see that only the resulting p-value for the Mortgage Loan of 0.048 is lower than our Acceptance Criteria of 0.05 then we will reject Null Hypothesis for this product and conclude that there is indeed significant difference between the late payment rates of those who received reminder and those who don’t.

For the other product, Baby Craving, Personal Loan, and Commodity Credit we will retain the Null Hypothesis and conclude that there is no significant difference between the late payment rates of those who received reminder and those who don’t.

<br>

___

# Statistical Modeling Considerations  <a name="concept-overview"></a>

Our assumptions on whether some other factors on top of the treatment (receiving or not receiving an email reminder) have an impact on the response (late payment behavior) guide the choice of statistical models:

Separate Analyses for Products: using a simple model like chi-square tests for each product can suffice, as we are primarily interested in assessing the effect of treatment across products, assuming no interaction with product types.

Complex Model with Demographis Characteristics: for Settlement Types and Age Group membership, a more complex model like log-linear model or logistic regression with interaction terms is necessary to accurately capture the joint effects and interactions between treatment and these other factors.

<br>
#### Separate Analyses for Products

As we already said, data were prodided in contingency tables, i.e. in matrix format. This format is perfect for applying Chi-Square Test For Independence. Chi-Square Test For Independence is a type of Hypothesis Test that assumes observed frequencies for categorical variables will match the expected frequencies.

Using a Chi-square Test for Interaction between Lateness and Treatment within Product: The chi-square test is appropriate when you want to examine the relationship between two categorical variables within the confines of a specific category (e.g., within each product). It is suitable when we assume that each product acts as a separate stratum and does not interact with other variables, thus allowing for a straightforward analysis of just the two variables of interest (lateness and treatment) within each product.

The *assumption* is the Null Hypothesis, which is always the viewpoint that the two groups will be equal.  With the Chi-Square Test For Independence we look to calculate a statistic which, based on the specified Acceptance Criteria will mean we either reject or support this initial assumption. The *observed frequencies* are the true values that we’ve seen. The *expected frequencies* are essentially what we would *expect* to see based on all of the data.

<br>
#### Complex Model with Demographis Characteristics

Using a Log-linear Model for Interaction between Lateness, Treatment, and Settlement Type/Age Group: This approach is needed when we are dealing with multiple categorical variables and are interested in understanding how these variables interact with each other across multiple dimensions. A log-linear model allows for the examination of complex interactions among three or more variables, giving insights into how combinations of factors influence an outcome, particularly when you suspect that these factors do not operate independently of each other.

___

<br>
# Data Overview & Preparation  <a name="data-overview"></a>

In this project, the contingency tables for treatment (reminder received or not) vs. outcome (late payment flag) plus product usage, settlement typy and age group were provided by data management team, thus importing data and creating contingency tables were already provided in matrix format.

___

<br>
# Applying Models: Focused Analysis vs. Joint Effect with Treatment <a name="applying-models"></a>

<br>
#### Focused Analysis: Product Choice - Treatment (Reminder Sent or Not Sent) Independence

In a campaign setup, if we assume that the response to the campaign (e.g., payment lateness) in the case of product choice depends only on the treatment (whether the customer is in the target group or control group), this implies that:

Product Homogeneity: The products are considered homogeneous in terms of how they influence customer behavior regarding payment lateness. This might be because the products are similar in nature, price, or the way they are marketed, making the treatment (target or control group) the primary factor that might alter behavior.

Focused Analysis: This allows for a focused analysis on how different treatments affect customer responses within each product. It simplifies the model by assuming no interaction between the product type and the treatment, which is often a valid assumption if previous data or market research indicates that the type of product does not modify the effect of the marketing treatment.


<br>
#### Focused Analysis within A Given Product Universe

We believe that the product choice has no effect on late payment behavior.  Therefoe, a simpler method, called Chi-Square Test For Independence is enough for us to ide if there is a difference in late payent between those who received a reminder 
 as opposed to those who did not. 
 
The fist step is to calculate the observed frequencies & expected frequencies. The *observed frequencies* are the true values that we’ve seen, in other words the actual rates per group in the data itself.  The *expected frequencies* are what we would *expect* to see based on *all* of the data combined.

The below code shows how to get the Chi-Square Test For Independence for all campaign particilant client and campaign participants within own product:

* We plug our data into the tests in a 2x2 matrix for *reminder_flag* (target groups vs, control group) by *payment_lateness_type* (response category)
* Based on this, the code below calculates the:
    * Chi-Square Statistic
    * p-value
    * Degrees of Freedom
    * Expected Values
* Prints out the Chi-Square Statistic & p-value from the test

```python

import numpy as np
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
all_clients = np.array([[11426, 2775],
                         [2785, 724]])

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
Based upon our observed values, we can give this all some context with the late payment rate of each group.  We get:

All Clients:
* Target Group (Reminder sent): **19.6%** late payment rate
* Control Group (Reminder NOT sent): **20.7%** late payment rate

Baby Craving:
* Target Group (Reminder sent): **10.9%** late payment rate
* Control Group (Reminder NOT sent): **12.5%** late payment rate

Personal Loan:
* Target Group (Reminder sent): **23.7%** late payment rate
* Control Group (Reminder NOT sent): **24.6%** late payment rate

Mortgage Loan:
* Target Group (Reminder sent): **2.2%** late payment rate
* Control Group (Reminder NOT sent): **3.2%** late payment rate

Commodity Credit:
* Target Group (Reminder sent): **46.5%** late payment rate
* Control Group (Reminder NOT sent): **54.8%** late payment rate


<br>
From this, we can see that being left our from the reminder campaign leads to a higher late payment for each loan product.  The results from our Chi-Square Test will provide us more information about how confident we can be that this difference is robust, or if it might have occured by chance: 


All Clients:
* Chi-square statistic: 2.12
* P-value: 0.146

Baby Craving:
* Chi-square statistic: 0.277
* P-value: 0.598

Personal Loan:
* Chi-square statistic: 1.11
* P-value: 0.292

Mortgage Loan:
* Chi-square statistic: 3.924
* P-value: 0.048

Commodity Credit:
* Chi-square statistic: 2.344
* P-value: 0.126

<br>
The critical value for our specified Acceptance Criteria of 0.05 is **3.84**

<br>
**Note** When applying the Chi-Square Test above, we use the parameter *correction = False* which means we are applying what is known as the *Yate's Correction* which is applied when your Degrees of Freedom is equal to one.  This correction helps to prevent overestimation of statistical signficance in this case.


<br> 
#### Joint Effect with Treatment - Settlement Type and Age Group

After assessing the effect of reminder sending on late payment behavior within the whole campaing base and within product types, we were also interested in seeing what impact the Settlement Type and Age Group of campaign participants had on the response output, late paymentbehavior. In other words, we need to know if Settlement Type and/or Age Group and treatment jointly affect the campaign response. The effectiveness of the treatment could vary by settlement type. For example, a promotional campaign might be more effective in urban areas than rural ones due to differences in access to payment facilities, economic activity, or consumer behavior. Or demographic characteristics may have inherent characteristics that significantly influence payment behaviors — such as economic conditions, demographic profiles, or logistical factors that affect how promotions or treatments are received.

___

<br>
# Analysing The Results <a name="chi-square-results"></a>

From the results above we can see that only the resulting p-value for the Mortgage Loan of **0.048** is *lower* than our Acceptance Criteria of 0.05 then we will _reject_ Null Hypothesis for this product and conclude that there is indeed significant difference between the late payment rates of those who received reminder and those who don't.

For the entire campaign base and all other products, Baby Craving, Personal Loan, and Commodity Credit we will _retain_ the Null Hypothesis and conclude that there is no significant difference between the late payment rates of those who received reminder and those who don't.

___

<br>
# Discussion <a name="discussion"></a>

While we saw that the recipients of payment reminder had a lower late payment rate (19.6%) than the control group (20.7%) it appears that this difference is not significant, at least at our Acceptance Criteria of 0.05. We got the same results within the universe of certain loan products, except Mongage Loans where the reminder had a significant impact on due payment bahavior.  

Without running this Hypothesis Test, the client may have concluded that they should always look to go with reminder emails - and from what we've seen in this test, that may not be a great decision.  It would result in them spending more for eDM, but not *necessarily* gaining any extra revenue as a result.
