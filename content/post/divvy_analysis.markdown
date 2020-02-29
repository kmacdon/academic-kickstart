---
title: "Predicting Divvy Bike Ride Durations"
summary: Using data from the Divvy bike share program in Chicago to predict trip duration
date: 2019-10-08
authors: ["admin"]
tags: ["r", "analysis", "regression"]
output: html_document
---




## Introduction

This analysis will look at the data for Divvy Bike rides in the city of Chicago during the year 2017. The data were collected from [Divvy's website](https://www.divvybikes.com/system-data) and merged with daily weather data from the [NOAA](https://www.ncdc.noaa.gov/cdo-web/datasets). The goal is to build a model which will predict the duration of the bike ride using the other variables contained in the data and interpet the effect of the different variables.

## Data 

I'll load the data and required packages as well as merge the Divvy data with the weather data by date.


```r
library(readr)
library(dplyr)
library(rsample)
library(magrittr)
library(ggplot2)
library(lubridate)

load('data/divvy/divvy.Rdata')
weather <- read_csv("data/divvy/weather.csv") %>% 
  select(DATE, PRCP, TAVG) %>% 
  mutate(PRCP = ifelse(PRCP > 0, T, F))

divvy$Date <- ymd(format(divvy$start_time, format = "%Y/%m/%d"))


divvy <- left_join(divvy, weather, by = c("Date" = "DATE"))
```

The dataset consists of about three million rides total once rows with NA values have been removed. 


|  trip_id|start_time          |end_time            | bikeid| tripduration| from_station_id|
|--------:|:-------------------|:-------------------|------:|------------:|---------------:|
| 13518905|2017-03-31 23:59:07 |2017-04-01 00:13:24 |   5292|          857|              66|
| 13518904|2017-03-31 23:56:25 |2017-04-01 00:00:21 |   4408|          236|             199|
| 13518903|2017-03-31 23:55:33 |2017-04-01 00:01:21 |    696|          348|             520|
| 13518902|2017-03-31 23:54:46 |2017-03-31 23:59:34 |   4915|          288|             110|
| 13518901|2017-03-31 23:53:33 |2017-04-01 00:00:28 |   4247|          415|             327|
| 13518900|2017-03-31 23:51:17 |2017-03-31 23:55:19 |   3536|          242|             143|

There are 19 variables in the dataset including the start and end times, start and end station names and ids, longitude and latitude for those stations, gender of the rider, their birthyear, an indicator for if there was precipitation, and the average temperature for the day along with some other ID variables I will not use. 

### Creating New Features
Before I start cleaning the data and exploring the distributions of the variables, there are a few other variables that should be created since they might be of use: rider age, month of ride, time of day the ride occurs, whether the ride is on a weekend or weekday, and the distance between stations in miles. 

Currently, I have the longitude and latitude of the stations, and need to convert this to a distance in miles. I'll use a modified version of the [Haversine Formula](https://en.wikipedia.org/wiki/Haversine_formula) to calculate the distance. Normally Haversine calculates the shortest distance between two geolocations, but that is not the best representation of the distance a bike rider travels since they have to follow city streets. I modified the formula to calculate the Manhattan distance instead since that will be a more accurate approximation of their route. 


```r
man_dist <- Vectorize(function(lon1, lon2, lat1, lat2){
  rad <- pi/180
  dlon <- abs(lon1 - lon2) * rad
  dlat <- abs(lat1 - lat2) * rad
  a <- sin(dlat/2)^2
  d <- 2*atan2(sqrt(a), sqrt(1-a))
  R <- 3963.19
  dlat <- R * d
  
  a <- sin(dlon/2)^2
  d <- 2*atan2(sqrt(a), sqrt(1-a))
  dlon <- R * d
  
  dlat + dlon
})
```

Now that I've created the function, I'll create the rest of the variables, including cutting the age up into bins.


```r
# Add all the new variables

divvy <- 
  divvy %>%  
  mutate(distance = man_dist(from_longitude, to_longitude, from_latitude, to_latitude),
         hour = hour(start_time),
         age = 2017 - birthyear,
         age_group = cut(2017 - birthyear, 
                         breaks = c(-Inf, 25, 35, 45, 55, 65, Inf),
                         labels = c("17-25", "26-35", "36-45", "46-55", "56-65", "66+")),
         month = factor(month(start_time)),
         time_of_day = cut(hour,
                           breaks = c(-1,8.5,16.5, 25),
                           labels = c("Morning", "Afternoon", "Evening")),
         weekend = ifelse(weekdays(start_time) %in% c("Saturday", "Sunday"), T, F))
```

### Data Cleaning

Before I start examining the data, it's important to clean it and remove some values that are clearly wrong. 
For one, the youngest rider is 0 years old and the oldest is 118 which does not make sense. I will remove all riders younger than 15 or older than 80, who only account for .07% of riders . Additionally, some rides start and end at the same station while others last for over an hour indicating either errors or unpredictable patterns. Again, I will remove these outliers, speficially any rides that have a distance of 0 or last longer than an hour which account for about 1.7% of the data with most of that coming from rides that have a distance of zero. 

These removals take out about 50,000 rides from the 3 million total. Going forward, I will split the data into a training and a test set using about 2/3 of the data for the training set.


```r
divvy <- 
  divvy %>% 
  filter(age > 15, age < 80, distance > 0, tripduration < 3600)

set.seed(101)
splits <- initial_split(divvy, prop = 2/3)

train_divvy <- training(splits)
test_divvy <- testing(splits)
```

### Exploratory Data Analysis
<img src="/post/divvy_analysis_files/figure-html/eda_plots-1.png" width="960" style="display: block; margin: auto;" />

These plots reaveal a lot of information. The trip duration and distance are both heavily skewed, even after removing trips longer than an hour. Most riders are in the age range 25-35 and rides spike during rush hour times which are 7am-9am and 4pm-6pm. More rides occur in the summer and fall than the winter, which makes sense since people are less likely to bike in cold weather. The average temperature looks to be high, but that makes sense since more rides occur in the summer than winter. About 80% of rides occur on weekdays (5/7 would be 72%), presumably because of commuting. Most rides occur on days without precipitation which isn't surprising, but it will be interesting to see what effect that variable has on trip duration.

### Plot Data Against Trip Duration
<img src="/post/divvy_analysis_files/figure-html/eda_cor-1.png" width="480" style="display: block; margin: auto;" />

There are some clear trends that stand out. The trip duration increases later in the day, but there are still clear spikes at rush hour. Since this isn't a linear relationship, I'll create a categorical feature that classifies rides into 3 groups based on what time of day they occured, midnight-8, 8-4, 4-midnight. Trip duration also increases during the summer months which may occur for a number of reasons. I'll create a new categorical feature to classify the rides as occuring in summer or winter to account for this. 

I also took a random sample of 5,000 points and made a scatter plot of temperature and trip duration. It shows that temperature does not seem to have much of an effect, since the smoothing line is nearly flat. The correlation between temperature and trip duration is only 0.123, so there doesn't seem to be any reason in keeping it. 

I'll look at the effects of Precipitation, Weekend, and Gender just by creating a table of the median duration for each value. 


|Variable |Value  | Med_Duration|
|:--------|:------|------------:|
|PRCP     |FALSE  |          576|
|PRCP     |TRUE   |          556|
|Weekend  |FALSE  |          557|
|Weekend  |TRUE   |          632|
|Gender   |Female |          648|
|Gender   |Male   |          547|

Trips are longer when there is no precipitation, but only by about 20 seconds which is not a significant distance. Trip length also increases on the weekend by about a minute which may be due to people not using the divvy bikes to commute. Women also have significantly longer trips, by about 100 seconds, so both gender and weekend have a noticeable impact. 


```r
# Create winter and summer variables
train_divvy$season <- ifelse(train_divvy$month %in% 4:9, "Summer", "Winter")

test_divvy$season <- ifelse(test_divvy$month %in% 4:9, "Summer", "Winter")
```

## Model Building

### First Model
I'll now build a model using gender, season, precipitation, time of day, and distance in order to predict trip duration.


```r
model <- lm(tripduration ~ distance + gender + PRCP + season + time_of_day, 
            data = train_divvy)
```

|                     | Estimate| Std. Error|  t value| Pr(>&#124;t&#124;)|
|:--------------------|--------:|----------:|--------:|------------------:|
|(Intercept)          |  212.751|      0.573|  371.533|                  0|
|distance             |  281.141|      0.140| 2005.800|                  0|
|genderMale           |  -64.099|      0.403| -159.045|                  0|
|PRCPTRUE             |  -10.118|      0.383|  -26.449|                  0|
|seasonWinter         |  -38.440|      0.372| -103.408|                  0|
|time_of_dayAfternoon |   61.904|      0.454|  136.235|                  0|
|time_of_dayEvening   |   52.494|      0.468|  112.201|                  0|

This model has an adjusted $ R^2 $ value of .68, which is fairly good for a problem such as this. Additionally, all of the features are statistically significant predictors of trip duration. 

In order to do some diagnostics on this model, I'll randomly select 5,000 residuals to plot. 

<img src="/post/divvy_analysis_files/figure-html/first_mod_res-1.png" width="672" style="display: block; margin: auto;" />


It's apparent that there is an issue with the residuals since they do not appear to have constant variance, and the distribution is skewed left. It would be interesting to see what the largest residual is.


|  distance| tripduration|
|---------:|------------:|
| 0.3493121|         3598|

The largest residual belongs to a trip that lasted almost an hour but only went .35 miles which does not make sense. It raises the question of how many outliers we should remove, so I'll examine the distribution of the speed of the trips in the training set in order to see if there are more data points that should be removed. 


<img src="/post/divvy_analysis_files/figure-html/unnamed-chunk-6-1.png" width="480" style="display: block; margin: auto;" />

| Min.| 1st Qu.| Median| Mean| 3rd Qu.|  Max.|
|----:|-------:|------:|----:|-------:|-----:|
| 0.12|    7.54|   9.22| 9.27|      11| 124.4|

Between the histogram and the summary of speed, it's obvious that there are issues with the data. For one, no one should be biking at speeds of 124 mph, but neither should they only be biking at .12 mph. For context, I used this [Wikipedia page](https://en.wikipedia.org/wiki/Bicycle_performance#Typical_speeds) describing typical cycling speeds. It explains that the low end should be about 6 mph while professional racing cyclists will average 25 mph, so to be conservative while still removing clear outliers, I'll remove all rides slower than 5 mph or faster than 20 mph. This removes about 5% of the points in the training set which is higher than I like, but necessary. The high speeds are clearly errors, but the low speeds could be people just taking leisurely, roundabout bike rides which would be very difficult to predict. 



### Second Model

```r
model <- lm(tripduration ~ distance + gender + PRCP + season + time_of_day, 
            data = train_divvy)
```
<img src="/post/divvy_analysis_files/figure-html/second_res-1.png" width="480" style="display: block; margin: auto;" />

Removing those points improved the $ R^2 $ to 0.81, but it's still clear from the residuals plot that the problems remain. Due to these issues, I will try to deal with the heteroscedascity by predicting the log of the duration using the log of the distance.

### Third Model

```r
train_divvy$log_dist = log(train_divvy$distance)
train_divvy$log_dur = log(train_divvy$tripduration)
test_divvy$log_dist = log(test_divvy$distance)
test_divvy$log_dur = log(test_divvy$tripduration)
```

```r
model <- lm(log_dur ~ log_dist + gender + PRCP + season + time_of_day, 
            data = train_divvy)
```
<img src="/post/divvy_analysis_files/figure-html/third_res-1.png" width="672" style="display: block; margin: auto;" />

Now, I'm are sacrificing interpretability for accuracy, but it looks like the tradeoff was worth it. The histogram of the residuals looks much more normal, and the scatterplot does not have the heteroscedascity of the earlier ones. Since the assumptions behind linear regression were not violated, there shouldn't be any issue interpreting the coefficients.


|                     | Estimate| Std. Error|   t value| Pr(>&#124;t&#124;)|
|:--------------------|--------:|----------:|---------:|------------------:|
|(Intercept)          |    6.041|      0.001| 12062.108|                  0|
|log_dist             |    0.858|      0.000|  3437.025|                  0|
|genderMale           |   -0.078|      0.000|  -197.989|                  0|
|PRCPTRUE             |   -0.005|      0.000|   -13.271|                  0|
|seasonWinter         |   -0.033|      0.000|   -90.948|                  0|
|time_of_dayAfternoon |    0.056|      0.000|   128.089|                  0|
|time_of_dayEvening   |    0.043|      0.000|    95.197|                  0|

Since the trip duration is now on a log scale, interpreting the effects of changes in the independent variables is now slightly more difficult, but [this](https://data.library.virginia.edu/interpreting-log-transformations-in-a-linear-model/) resource is helpful for the task.

The coefficient for the log distance was about .858, so this means that for every 10% increase in distance, the expected trip duration will increase by about 8.5%, clearly a strong relationship. All else being equal, a man would be expected to have a trip duration that is about 7.5% less than a woman's. 

Precipitation had a very negligible effect while winter decreased durations by about half as much as being male does, 3.2 % . Interestingly, the time of day being in the afternoon increased the expected duration by about 5.7% with evening commutes having a similar one. Perhaps the time of day could have been incorporated better by using more cut points to discretize it or even fitting something like cubic splines, but I'll keep it simple and just use this model.

The adjusted $ R^2 $ is about 0.87, so this model explains about 87% of the variation in tripduration (for the dataset without outliers). Just to see if I'm overfitting the data, I'll compare the mean squared error and mean absolute percentage error of the training data to the test data, which did not have the speed outliers removed. I also converted the predictions back from the log scale to the regular scale so that the results can be interepreted more easily.


|          |           |         |          |
|:---------|:----------|:--------|:---------|
|MSE Train |MAPE Train |MSE Test |MAPE Test |
|30133.24  |18.61      |61076.92 |20.71     |

Our model performed better on the training data then the test data, which was to be expected in any scenario but especially one in which I removed outliers in one set and not the other. The difference in MSE is very large, but considering the outliers in the test data set would produce very large residuals, and MSE would penalize those more heavily, this is not too big of an issue. The MAPE for both sets is fairly comparable, showing that on average, our durations were off by about 21% on the test data, an acceptable result.

## Conclusions

This is a large data set with many variables, so one could spend weeks analyzing the data and doing feature selection and engineering to come to a good result. My approach was simpler, and although I risked fitting the data to the model by removing outliers, those removals were necessary since the data does have outliers that simply do not make sense. The end result is a satisfactory model for trip duration and showed that factors such as gender, time of day, and season were all important in determining the length of the rides. 
