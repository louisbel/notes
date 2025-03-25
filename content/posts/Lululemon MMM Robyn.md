---
title: "Exploring Media Mix Modeling with Robyn and GPT-4o"
date: "2024-05-28"
summary: "Can GPT assist with Media Mix modeling in R?"
toc: false
readTime: true
autonumber: false
math: true
---
Code and statistical analysis are often integral to a marketer's work. Many of us tinker with JavaScript, updating variables in Google Ads scripts, debugging Analytics implementations, examining source code, and manipulating large datasets for reporting.

However, most of us, myself included, are not developers or statisticians. When OpenAI released GPT-4o earlier this month, I was impressed by the positive debugging feedback on Reddit and wondered if it could help bridge the technical gap needed to implement **[Robyn](https://facebookexperimental.github.io/Robyn/), Meta’s open-source package for Media Mix Modeling**. Although Robyn had been on my radar for a couple of years, the setup instructions always seemed daunting.

## Why Media Mix Modeling?

HBR has an insightful article on the importance of calculating channel contributions to ROI.

> *"(MMMs) have a specific advantage: They’re able to produce dependable measurements — and insight — purely from natural variation in aggregate data, and don’t require user-level data."* - [A New Gold Standard for Digital Ad Measurement?](https://hbr.org/2023/03/a-new-gold-standard-for-digital-ad-measurement) by Julian Runge, Harpreet Patter, and Igor Skokan

---
## Building the Dataset for Robyn

![image](/notes/images/MMM_Dataset2.gif)

For the sake of this experiment[^1], I built an hypothetical dataset (2011 to 2013) for clothing brand Lululemon using some publicly available data:

- Annual US revenue & global marketing expenses from **[the company's 10K-filings](https://corporate.lululemon.com/investors/financial-information/annual-reports)**
- The **[Retail Seasonality Index](https://www.census.gov/retail/marts/www/adv44800.txt)** from Census.gov to allocate revenue based on weekly seasonality in the retail industry
- Direct to consumer revenue = 45% of total revenue (disclosed in the 2022 annual report)

And multiple assumptions:
 - Lululemon's share of US marketing expenses is proportional to the share of US revenue on global revenue (85% in 2021, 79% in 2023)
 - Its media costs account for 70% of total marketing expenses (176M in 2021, 239M in 2023)
 - The company's media mix is split between Meta (25%), Google (24%),  TV (17%), OOH (13%), TikTok (11%), Print (7%), Pinterest (3%) and Email
 - Ad inventory cost averages have trended similarly to monthly global channel benchmarks
 - The company has 9 important sales each year (ie. end of season sales and Cyber Monday)

The Robyn guidelines recommend segmenting the largest media channels to further expose the performance of the underlying tactics. I split the Google and Meta spend between prospecting and retargeting budgets. I then introduced some noise in the weekly data by randomizing media spend and performance metrics within a +1% to -1% range for each weekly value.

---
## How GPT-4o helped me bridge the gap

Throughout the experiment, I encountered several issues where GPT-4o was a helpful companion.

### Debugging errors when installing the Robyn package

Robyn runs in R and requires additional resources to be installed such as [Nevergrad](https://facebookresearch.github.io/nevergrad/), an open-source Python library. When trying to install this resource, I got an error where an important dependency "cmake" was not installed on my computer, therefore Nevergrad could not be installed.

GPT guided me through the multiple steps to remediate the issue, including installing cmake.

```r
Lou@LB1 ~ % brew install cmake
```

### Getting feedback on running & calibrating the model

The complete set of instructions is available within the official Robyn demo file on [Github](https://github.com/facebookexperimental/Robyn/blob/main/demo/demo.R).

**Step 1: Collecting all the inputs**

This step is about feeding the context of the model to Robyn. GPT was helpful to validate the InputCollect function and contextualize any errors that appeared in the R console, mostly due to formatting discrepencies between my naming conventions for channels.

```r
InputCollect <- robyn_inputs(
  dt_input = lululemon_v16,
  dt_holidays = dt_prophet_holidays,
  date_var = "DATE", # date format must be "2020-01-01"
  dep_var = "revenue", # there should be only one dependent variable
  dep_var_type = "revenue", # "revenue" (ROI) or "conversion" (CPA)
  prophet_vars = c("trend", "season", "holiday"), # "trend","season", "weekday" & "holiday"
  prophet_country = "US", # input country code. Check: dt_prophet_holidays
  context_vars = c("competitor_sales_B", "events"), # e.g. competitors, discount, unemployment etc
  paid_media_spends = c("tv_S","ooh_S","print_S","facebook_S_P","facebook_S_R","search_S_P","search_S_R","TikTok_S","Pinterest_S"), # mandatory input
  paid_media_vars = c("tv_S","ooh_S","print_S","facebook_I_P","facebook_I_R", "search_C_P","search_C_R","TikTok_I","Pinterest_I"), # mandatory.
  # paid_media_vars must have same order as paid_media_spends. Use media exposure metrics like
  # impressions, GRP etc. If not applicable, use spend instead.
  organic_vars = "newsletter", # marketing activity without media spend
  # factor_vars = c("events"), # force variables in context_vars or organic_vars to be categorical
  window_start = "2021-01-04",
  window_end = "2023-12-25",
  adstock = "geometric" # geometric, weibull_cdf or weibull_pdf.
)
print(InputCollect)
```

**Step 2: Setting the hyperparameters**

This is where we set the *Adstock* (the carryover effect from marketing) and the *Saturation* (the diminishing returns of marketing) with the 3 hyperparameters. Robyn recommends setting greater decay for digital than offline channels. I used the default values from the demo file.

```r
hyper_names(adstock = InputCollect$adstock, all_media = InputCollect$all_media)

hyperparameters <- list(
  facebook_S_P_alphas =c (0.5, 3),
  facebook_S_P_gammas =c (0.3, 1),
  facebook_S_P_thetas =c (0, 0.3),
  facebook_S_R_alphas =c (0.5, 3),
  facebook_S_R_gammas =c (0.3, 1),
  facebook_S_R_thetas =c (0, 0.3),
  newsletter_alphas =c (0.5, 3),
  newsletter_gammas =c (0.3, 1),
  newsletter_thetas =c (0.1, 0.4),
  ooh_S_alphas =c (0.5, 3),
  ooh_S_gammas =c (0.3, 1),
  ooh_S_thetas =c (0.1, 0.4),
  Pinterest_S_alphas =c (0.5, 3),
  Pinterest_S_gammas =c (0.3, 1),
  Pinterest_S_thetas =c (0, 0.3),
  print_S_alphas =c (0.5, 3),
  print_S_gammas =c (0.3, 1),
  print_S_thetas =c (0.1, 0.4),
  search_S_P_alphas =c (0.5, 3),
  search_S_P_gammas =c (0.3, 1),
  search_S_P_thetas =c (0, 0.3),
  search_S_R_alphas =c (0.5, 3),
  search_S_R_gammas =c (0.3, 1),
  search_S_R_thetas =c (0, 0.3),
  TikTok_S_alphas =c (0.5, 3),
  TikTok_S_gammas =c (0.3, 1),
  TikTok_S_thetas =c (0, 0.3),
  tv_S_alphas =c (0.5, 3),
  tv_S_gammas =c (0.3, 1),
  tv_S_thetas =c (0.3, 0.8),
  train_size = c(0.5, 0.8)
)
```

GPT helped contextualize the distribution of the parameters.

![image](/notes/images/MMM_hypersampling-GPT1.png)  
![image](/notes/images/MMM_hypersampling-GPT2.png)

**Step 3: Running all trials**

This is where Robyn runs multiple trials to see how well different combinations of media mix affect the revenue outcome.

```r
OutputModels <- robyn_run(
  InputCollect = InputCollect, # feed in all model specification
  cores = NULL, # NULL defaults to (max available - 1)
  iterations = 2000, # 2000 recommended for the dummy dataset with no calibration
  trials = 5, # 5 recommended for the dummy dataset
  ts_validation = TRUE, # 3-way-split time series for NRMSE validation.
  add_penalty_factor = FALSE # Experimental feature. Use with caution.
)
print(OutputModels)
```

![image](/notes/images/MMM_R.gif)

**Step 4: Calculating Pareto Fronts**

Robyn then dentifies the optimal trade-offs among different objectives (e.g., ROI vs. cost) using Pareto efficiency.

```r
OutputCollect <- robyn_outputs(
  InputCollect, OutputModels,
  pareto_fronts = "auto", # automatically pick how many pareto-fronts to fill min_candidates (100)
  # min_candidates = 100, # top pareto models for clustering. Default to 100
  # calibration_constraint = 0.1, # range c(0.01, 0.1) & default at 0.1
  csv_out = "pareto", # "pareto", "all", or NULL (for none)
  clusters = TRUE, # Set to TRUE to cluster similar models by ROAS. See ?robyn_clusters
  export = create_files, # this will create files locally
  plot_folder = robyn_directory, # path for plots exports and files creation
  plot_pareto = create_files # Set to FALSE to deactivate plotting and saving model one-pagers
)
print(OutputCollect)
```

After this step, Robyn shows the contribution of each channel to overall revenue and the estimated ROAS. Some channels have a greater contribution to revenue (Effect Share) for their budget (Spend Share). In this case TikTok (7.28 ROAS) and prospecing campaigns on Search (7.04 ROAS) are most efficient. 

![image](/notes/images/MMM_ROAS.png)

**Step 5: Re-allocating the budgets**

Here we select a specific model and have Robyn generate new budgets for each channels. We're able to set constraints such as the maximum budget available and how much we allow a channel spend to flucutate. In this case no less than 70% of the current budget but no more than 210%.

```r
select_model <- "1_230_2"
ExportedModel <- robyn_write(InputCollect, OutputCollect, select_model, export = create_files)
print(ExportedModel)
InputCollect$paid_media_spends

AllocatorCollect1 <- robyn_allocator(
  InputCollect = InputCollect,
  OutputCollect = OutputCollect,
  select_model = select_model,
  # date_range = "all", # Default to "all"
  # total_budget = NULL, # When NULL, default is total spend in date_range
  channel_constr_low = 0.7,
  channel_constr_up = 2.1,
  # channel_constr_multiplier = 3,
  scenario = "max_response",
  export = create_files
)
```

The Robyn model shows that based on the hypothetical data, reallocating media spend from Pinterest, Meta's prospecting campaigns, OOH and TV towards Search and Print could increase revenue and that multiple channels are not yet saturated with their current budget.

![image](/notes/images/MMM_Response-Curve.png)
![image](/notes/images/MMM_Budget-Allocation.png)

I found GPT particularly useful for debugging the R code when the error returned in the console lacked context for troubleshooting. It was also insightful with knowledge gaps around statistics and the interpretation of the output. This experiment was valuable for unpacking the components of building a a media mix model and gaining a better understanding of the requirements and limitations.

Overall, Robyn is a very advanced package with many more features than demonstrated here. Media Mix Modeling is a long-term endeavor that requires thoughtful rounds of calibration, which are beyond the scope of this experiment.

[^1]: There is an open source package I haven't tested called SiMMMulator to help generate simulated data to plug into Marketing Mix Models (MMMs).