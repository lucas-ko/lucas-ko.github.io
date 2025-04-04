---
layout: post
title: Azure Infrastructure - Log Analytics - calculate daily ingested data with moving average
subtitle: 
date: 2024-07-11T19:00:00.000Z
tags:
  - Azure Infrastructure
  - Operational Excellence
  - Cost Efficency
  - FinOps
nav-short: true
author: Lukasz Kozubal
---

If you need to get information about the size of billable data ingested into given Log Analytics workspace (with or without Sentinel solution installed), use the following KQL query:

```
//configure lookback period
let lookback = 90d;
Usage
//include only billable data and remove current day in time filter
| where TimeGenerated between(ago(lookback)..now(-1d)) and IsBillable == true
| project TimeGenerated, Quantity
//create series representing data ingested per day (converted from MB to GB, change divisor to 1024 if you prefer binary definition of gigabyte (GiB))
| make-series DailyIngestionGB=sum(Quantity/1000) default=0 on TimeGenerated step 1d
//calculate moving average with FIR function using fixed size filter (5) of equal coefficients
| extend MovingAvg = series_fir(DailyIngestionGB,repeat(1, 5))
| project TimeGenerated, DailyIngestionGB, MovingAvg
//render on a line graph, X-axis represent time points, Y-axes represent sum of ingested data in a given day and calculated moving average per that day
| render timechart with (xtitle = "Date", ytitle = "Ingested data (GB)")
```
As an output you will receive line chart with two Y-axes lines representing:
- Sum of ingested, billable data in a given day.
- Moving average calculated using [series_fir()](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/series-fir-function) (finite impulse response) function. _Series_fir()_ calculates rolling average over a constant time window (one day in this case) on the input dataset. Using moving average enables you to see the trend more clearly by filtering out the noise from ingestion fluctuations (e.g. on the weekends).

![347875709-e8a0f13e-a555-425a-b03f-8ebdf22c56d8](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/bfeb9b06-1433-4a1c-88b8-455c9d7e064c)

## Helpful sources
- [Analyze usage in a Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/analyze-usage)
- [Usage table - schema reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/usage)

---
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
