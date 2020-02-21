---
title: "Fantasy Football Analysis"
summary: An analysis of my disappointing fantasy football season using Python
date: 2019-12-13
authors: ["admin"]
tags: ["python", "analysis", "simulation"]
output: html_document
---




```python
import pandas as pd
import matplotlib.pyplot as plt
from plotnine import *
import numpy as np
```


## Introduction

After a disappointing fantasy football season finishing 6th out of 10 teams, I decided to analyze the season and see what information I could gather from the weekly scores and possibly show that I had a better performance than the final rankings indicated.

## Data 


```python
scores = pd.read_csv("data/fantasy/fantasy_scores.csv")
scores.head()
```

```
##    Week   Tony  Shaker  Quinn     Up  ...  Healthy  Quindawg   Heff    Win  Lamar
## 0     1  114.0   104.5  154.0  107.5  ...    100.5     100.0  100.0  140.0  111.0
## 1     2   97.0   132.5  110.0   99.0  ...     61.5      82.5   85.5  113.5   97.5
## 2     3   92.0   114.0  138.0  126.5  ...    112.0      89.5   80.5  128.5  129.0
## 3     4   99.5   112.0   38.5   74.0  ...    127.5      78.0   97.5  132.0  108.0
## 4     5  118.0   137.5  103.5  105.0  ...    126.0     118.5  147.0  167.0   88.0
## 
## [5 rows x 11 columns]
```

I collected the scores for each team for each week by hand and stored them in a csv file. My team is Shaker in this data set, short for Shaker and Baker due to my misplaced hopes in Baker Mayfield. 


```python
melt_scores = pd.melt(scores, id_vars = ['Week']) \
                .rename(columns = {"variable":"Team", "value":"Score"})
                
melt_scores.head()
```

```
##    Week  Team  Score
## 0     1  Tony  114.0
## 1     2  Tony   97.0
## 2     3  Tony   92.0
## 3     4  Tony   99.5
## 4     5  Tony  118.0
```

It will be easier to analyze and plot this data by putting it in long format. I'll start off by plotting the total points scored for the season.

## Analysis


```python
totals = melt_scores.groupby('Team').agg({'Score':sum}).sort_values('Score')
```

<img src="/post/fantasy_analysis_files/figure-html/unnamed-chunk-5-1.png" width="614" style="display: block; margin: auto;" />

As you can see, I scored the 3rd most points in the regular season, already evidence that perhaps my team deserved a better finish than it received. My conference had three of the top four scoring teams which definitely impacted my record. 


```
## <ggplot: (-9223372029287213508)>
```

<img src="/post/fantasy_analysis_files/figure-html/unnamed-chunk-6-1.png" width="614" style="display: block; margin: auto;" />

This is the way the results actually turned out. The top four teams made the playoffs while some high scoring teams managed to finish in the bottom half of the league.


<img src="/post/fantasy_analysis_files/figure-html/unnamed-chunk-8-1.png" width="614" style="display: block; margin: auto;" />

If we look at box plots for each team, we can see that my team had the second highest median weekly score out of all the teams. Not only that, my team was fairly consistent, especially compared to some teams which were heavily skewed in the wrong direction.

## Simulation

My 6th place finish was not due to my teams perfomance, but to the tough schedule I faced. I was in the far stronger conference of the two, and that meant that in many weeks an above average performance was simply not enough. 

In order to see how this possibly could have turned out, I'll run a simulation of 1000 seasons. The way schedules are decided is that each team plays every other team in its conference twice and every team in the other conference once. I'll randomly select conferences, then randomly select teams to fill out my schedule according to this rule and see how often I do better than just 6 wins.


```python
def simulation(scores, teams, n=1000):
    season_wins = []
    for i in range(n):
        conferences = np.isin(teams, np.random.choice(teams, 5, replace=False))
        my_conference = conferences[np.where(teams == 'Shaker')]
        schedule = np.random.permutation(np.concatenate((teams[np.logical_and(conferences == my_conference, teams != 'Shaker')], 
                                                         teams[np.logical_and(conferences == my_conference, teams != 'Shaker')], 
                                                         teams[conferences != my_conference])))
        wins = 0
        for week, team in enumerate(schedule):
            if scores['Shaker'].iloc[week] > scores[team].iloc[week]:
                wins += 1
        season_wins.append(wins)
    return season_wins
```


```python
np.random.seed(101)
teams = np.array(scores.columns[1:].tolist())
total_wins = simulation(scores, teams)
```
<img src="/post/fantasy_analysis_files/figure-html/unnamed-chunk-11-1.png" width="614" style="display: block; margin: auto;" />

Already the histogram shows that getting just 6 wins in a season was a rare occurence. To get more specific probabilities, I'll look at the summary statistics of the total wins from this simulation.


```python
np.quantile(total_wins, np.linspace(0, 1, 5))
```

```
## array([ 3.,  7.,  8.,  9., 12.])
```

Even my 25th percentile season had more than 6 wins, and in 50% of the seasons I had at least 8 wins! Clearly, my team performed well, it just had unfortunate scheduling. This shows that having the concept of weekly games in fantasy football doesn't make sense. Yes, the goal is to imitate the NFL, but unlike an actual NFL game, I have no control over how many points my opponent scores. Fantasy football is more similar to a race; each person can only control how well they perform, and they should be evaulated based on how well they perform against the field, not on how well they perform against any one opponent. Long story short is I'm bitter.
