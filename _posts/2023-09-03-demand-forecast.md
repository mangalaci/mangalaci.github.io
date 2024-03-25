---
layout: post
title: Demand Forecast of Retail Products Using ARIMA Method
image: "/posts/demand-forecast-arima.png"
tags: [Demand Forecast, ARIMA, Time Series, Python]
---

In this comprehensive analysis, I embarked on a journey to enhance the inventory management and sales strategy of eOptika, a European international optical product retailer and webshop chain. By leveraging sophisticated statistical and machine learning methods, I aimed to accurately forecast product demand mostly in order to optimize investment in stock inventory. At the first phase of the project, I focused on product level (instead of product group, brand or manufacturer level). The forecast spanned from January 2018 to February 2024, providing invaluable insights for strategic planning and operational efficiency.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Context](#overview-context)
    - [Actions](#overview-actions)
    - [Results & Discussion](#overview-results)
- [01. Data Acquisition & Preprocessing](#data-overview)
- [02. Forecasting Models Implementation](#chi-square-application)
- [03. Analysing The Results](#chi-square-results)
- [04. Discussion](#discussion)

___

# Project Overview  <a name="overview-main"></a>

### Context <a name="overview-context"></a>

In the rapidly evolving retail environment, eOptika faced challenges in maintaining optimal stock levels across its stores nationwide. Excess inventory leads to increased holding costs, while stockouts risk losing sales and customer satisfaction. Addressing this challenge, our project focuses on forecasting the demand for the whole product line, which has shown variable sales patterns over recent years.


<br>
<br>
### Actions <a name="overview-actions"></a>

To develop a reliable demand forecasting model that predicts monthly sales for the "CT2_pack" product line.
To provide actionable insights for inventory management and sales strategies, reducing holding costs and minimizing stockouts.



<br>
<br>

### Methodology Overview <a name="methodology-overview"></a>

Utilizing Python's sqlalchemy library, I performed data extraction from eOptika's MySQL database, followed by data preprocessing to ensure quality and consistency. I then explored various forecasting models, namely AutoARIMA and ETS, and LSTM to capture the underlying sales trends and seasonality. The models were evaluated based on Root Mean Squared Error (RMSE), guiding us to the best-fit model for eOptika.

___

<br>**
# Data Acquisition & Preprocessing  <a name="data-overview"></a>**

The project commenced with establishing a secure connection to eOptika's analytics database, from which we extracted historical sales data on a product level product line. Since this is a univariate analysis, we only need a time variable, in our case it is  "month" and the measure that we want to forecast, "items_sold (units)".

<br>
A sample of this data (the first 5 rows) can be seen below:
<br>
<br>

| **product** | **items_sold (units)** | **month** |
|---|---|---|---|
| 1 Day Acuvue Moist (30 db) | 2018-01 | 498 |
| 1 Day Acuvue Moist (30 db) | 2018-02 | 387 |
| 1 Day Acuvue Moist (30 db) | 2018-03 | 454 |
| 1 Day Acuvue Moist (30 db) | 2018-04 | 473 |
| 1 Day Acuvue Moist (30 db) | 2018-05 | 539 |

<br>

___


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
