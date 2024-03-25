---
layout: post
title: Demand Forecast of Retail Products Using ARIMA Method
image: "/posts/demand-forecast-arima.png"
tags: [Demand Forecast, SARIMA, Time Series, Exponential Smoothing, LSTM, Python]
---

In this comprehensive analysis, I embarked on a journey to enhance the inventory management and sales strategy of eOptika, a European international optical product retailer and webshop chain. By leveraging sophisticated statistical and machine learning methods, I aimed to accurately forecast product demand mostly in order to optimize investment in stock inventory. At the first phase of the project, I focused on product level (instead of product group, brand or manufacturer level). The forecast spanned from January 2018 to February 2024, providing invaluable insights for strategic planning and operational efficiency.

# Table of contents

- [00. Project Overview](#overview-main)
    - [Context](#overview-context)
    - [Actions](#overview-actions)
    - [Methodology Overview](#overview-methodology)
- [01. Data Acquisition & Preprocessing](#data-overview)
- [02. Forecasting Models Implementation](#forecast-model-implement)
- [03. Discussion](#discussion)

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

### Methodology Overview <a name="overview-methodology"></a>

Utilizing Python's sqlalchemy library, I performed data extraction from eOptika's MySQL database, followed by data preprocessing to ensure quality and consistency. I then explored various forecasting models, namely AutoARIMA and ETS, and LSTM to capture the underlying sales trends and seasonality. The models were evaluated based on Root Mean Squared Error (RMSE), guiding us to the best-fit model for eOptika.

___

<br>
# Data Acquisition & Preprocessing  <a name="data-overview"></a>

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

# Forecasting Models Implementation <a name="forecast-model-implement"></a>

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
