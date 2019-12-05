---
title: "Suicide Rates, Population Density, and Gun Ownership"
summary: Analyze the relationship between suicide rate, population density, and gun ownership at the state level
date: 2019-12-04
authors: ["admin"]
tags: ["r", "analysis"]
output: html_document
---



## Introduction

I decided to do this analysis after seeing a chart on gun violence in the United States by state. It listed out both homocide and suicide rates for each state, and I noticed that the states with high gun suicide rates tended to be more sparsely populated and wanted to explore this further. I acquired the 2017 suicide rate in deaths per 100,000 people for each state from the [CDC](https://www.cdc.gov/nchs/pressroom/sosmap/suicide-mortality/suicide.htm) and the 2019 population density in people per square mile for each state from [worldpopulationreview.com](http://worldpopulationreview.com/states/state-densities/). I couldn't find the 2017 density estimates, but I assume there won't have been much change in 2 years time. I also acquired the proportion of gun owners for each state for 2019 from [worldpopulationreview.com](http://worldpopulationreview.com/states/gun-ownership-by-state/), although it should be noted that this is a difficult statistic to estimate. 

## Data Exploration


```r
library(dplyr)
library(ggplot2)
library(maps)
library(fiftystater)

suicides <- readr::read_csv("state_suicide.csv") %>% 
  filter(YEAR == 2017) %>% 
  select(STATE, RATE)
```

```r
# Merge data with map information for plotting
state_names <- tibble(abbrev = state.abb,
                          names = state.name)
suicides <- left_join(suicides, state_names, by = c("STATE" = "abbrev"))

suicides$names <- stringr::str_to_lower(suicides$names)
```
<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-3-1.png" width="672" style="display: block; margin: auto;" />

It's immediately noticeable that Montana and Wyoming have the highest suicide rates in the country, and states in the Great Plains and Rocky Mountain regions generally have higher rates than other areas of the country.


```r
pop <- readr::read_csv("population.csv") %>% 
  select(State, Density) %>% 
  mutate(State = stringr::str_to_lower(State))
```
<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-5-1.png" width="672" style="display: block; margin: auto;" />

I used the log of the population density so that the differences among states can be viewed more clearly. If we look at Montana and Wyoming again, we can see that they again stand out, this time as having the lowest population density. In general, this plot is close to the inverse of the previous one, which shows that there is some level of correlation between these two variables.


```r
guns <- readr::read_csv("gun_ownership.csv") %>% 
  select(State, gunOwnership) %>% 
  mutate(State = stringr::str_to_lower(State))
```
<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-7-1.png" width="672" style="display: block; margin: auto;" />

The plot of the proportion of gun owners by state shows similar trends to the two previous plots, so now I'll move on to comparing these variables to each other directly.

## Relationship


```r
full_data <- left_join(suicides, pop, by = c("names" = "State")) %>% 
  left_join(., guns, by = c("names" = "State"))
```


<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-9-1.png" width="480" style="display: block; margin: auto;" />

The correlation between density and suicide rates is extremely apparent from this graph, although the relationship is non-linear. In order to make this relationship more linear, I'll take the natural log of the density and plot the data again.

<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-10-1.png" width="480" style="display: block; margin: auto;" />

The improvement is clear, and it is easy to see that there is a strong correlation between these two variables. Now I'll look at the relationship between gun ownership and suicide.

<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-11-1.png" width="480" style="display: block; margin: auto;" />

Again, there is a strong and obvious correlation between these two variables, which also means there is a strong correlation between gun ownership and population density.

<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-12-1.png" width="480" style="display: block; margin: auto;" />

This multicollinearity will be an issue as I move into building a model using these variables. 

## Model Building

#### Density

```r
mod_density <- lm(RATE ~ log(Density), data = full_data)
```

|             | Estimate| Std. Error| t value| Pr(>&#124;t&#124;)|
|:------------|--------:|----------:|-------:|------------------:|
|(Intercept)  |   30.424|      1.095|  27.795|                  0|
|log(Density) |   -3.015|      0.230| -13.099|                  0|

As expected, the log density is extremely significant, and the R^2^ value is about 0.781. I'll also examine the residuals of the model to make sure nothing is unusual.

<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-15-1.png" width="768" style="display: block; margin: auto;" />

The histogram of the residuals appears to be approximately normal, although there is a noticeable gap in the data. The scatterplot does not reveal any problems with the variance of the residuals, so it does not appear any assumptions were violated. 

#### Gun Ownership

```r
mod_gun <- lm(RATE ~ gunOwnership, data = full_data)
```

|             | Estimate| Std. Error| t value| Pr(>&#124;t&#124;)|
|:------------|--------:|----------:|-------:|------------------:|
|(Intercept)  |    8.477|      1.278|   6.635|                  0|
|gunOwnership |   24.854|      3.578|   6.946|                  0|

Gun ownership is also a significant predictor of suicide rate, but the R^2^ value is about 0.501, lower than the previous model.

<img src="/post/suicide_analysis_files/figure-html/unnamed-chunk-18-1.png" width="768" style="display: block; margin: auto;" />

The scatterplot of the residucals doesn't show any pattern, but the histogram is clearly not normally distributed. Combined with the lower R^2^ value, I would conclude that population density is a better predictor of suicide rate than gun ownership. 

As mentioned before, there is the issue of multicollinearity which means I can't just build a model using both gun ownership and population density as predictors. I could use principal component analysis in order to eliminate this issue, but that would sacrifice interpretability which I think is more important in this situation, so I won't continue with model building.

## Conclusion

The plots and models show that population density is a strong predictor of suicide rate, explaining about 78% of the variability. As it's always important to remember, correlation does not equal causation. Suicide is not a simple issue to explain, and there are countless factors that are involved. However, the strong correlation between these variables, and gun ownership, are interesting areas to explore, especially since about half of all suicides involve firearms^[https://www.cdc.gov/nchs/fastats/suicide.htm].
