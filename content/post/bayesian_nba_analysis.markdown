---
title: Bayesian Analysis of NBA Winning Percentages
date: 2018-10-01
summary: A simple modeling project using Bayesian statistics for NBA teams
authors: ["admin"]
tags: ["r", "analysis"]
output: html_document
---



## Introduction
This project uses Bayesian analysis to look at the winning percentages of the Cleveland Cavaliers and the Golden State Warriors. Using data from 2013 through 2017, I will analyze the posterior distribution in order to make predictions about the 2018 season. I use a prior of Beta(50,50) for most of the analysis although some sections do look at the effects of using a different prior. The likelihood function follows a Binomial distribution since I am examining Win/Loss data. 

These two distributions are conjugate distributions, so my posterior distrubtion for the winning percentage will be Beta as well, with the parameters just being the prior alpha and beta plus the number of wins and loses respectively.

<img src="/post/bayesian_nba_analysis_files/figure-html/unnamed-chunk-1-1.png" width="672" style="display: block; margin: auto;" />

The only data used in this analysis is the number of wins for the Warriors and Cavaliers for each season from 2012 to 2017. The two teams have followed similar paths; although the Warriors had more wins in every season, both teams saw a large increase in wins starting with the 2014-2015 season. 

## Analysis
### Cavaliers: All Seasons

<img src="/post/bayesian_nba_analysis_files/figure-html/unnamed-chunk-3-1.png" width="672" style="display: block; margin: auto;" />

To begin, I'll examine the posterior distribution for the Cavalier's expected winning percentage for this season using data from their previous five seasons during which time they won 24, 33, 53, 57, and 51 games as well as one NBA championship. Fig. 1 shows the changes in the posterior distributions as more seasons are included, i.e the plot for the 2012-2013 season shows the posterior distribution once the record from that season is included in the analysis. During the Cavs two seasons from 2012-2014, they had a losing record, which is reflected in the posterior distributions for those seasons since they are centered around .40. The next season, the Cavs ended up winning 53 games due to the return of LeBron James, and it can be seen that the posterior distribution shifted right, from a mean of .40 to a mean of .46 while the density of the distribution continues to become more centered around the mean. As the Cavs continued to play well, the distribution continued to shift right, with the final posterior distribution having a mean of .525. This translates to an expectation of about 43 wins for the Cavs and a a 95% chance of winning between 39.5 and 46.6 games in the current season. Clearly, that would be a surprising drop off considering three straight seasons of at least 50 wins. This is due to the inclusion of the losing seasons before Lebron James return to Cleveland, which pull the distrubtions to the left.

### Cavaliers: Reduced Seasons

<img src="/post/bayesian_nba_analysis_files/figure-html/unnamed-chunk-5-1.png" width="672" style="display: block; margin: auto;" />

In order to find a more accurate posterior distribution, I removed the two outlier seasons and changed the prior to Beta(65,50) - this has a mean of .565 - since the Cavs would be expected to win over half their games with LeBron James considering the enormous impact a player of his caliber can have on one team. Fig. 2 shows this shifted the posterior distributions much further right than before, resulting in the final distribution having a mean of .626. This would translate to 51 wins for the Cavs, an amount consistent with their previous performance. This analysis gives them a 95% chance of winning between 47.2 and 55.4 games in the 2018 season. Currently the Cavs are on pace to win about 48 games this year after starting off poorly. This prediction may be overconfident however due to the trade of Kyrie Irving, the Cavaliers' second best player, to Boston; it is unclear if the Cavs will be able to replace his production.

### Warriors: All Seasons

<img src="/post/bayesian_nba_analysis_files/figure-html/unnamed-chunk-7-1.png" width="672" style="display: block; margin: auto;" />

In order to compare the Cavs to another team, I also performed a Bayesian analysis on the winning percentage of the Golden State Warriors, since they have been the best team in the league for multiple seasons. Additionally, they have not experienced a change like the addition of Lebron to Cleveland (the Warriors were already NBA champions before the addition of Kevin Durant while the Cavaliers were one of the worst teams in the league before LeBron James' return). Over the previous five seasons, the Warriors have won 47, 51, 67, 73, and 67 games, including winning 2 NBA championships. As can be seen in Fig. 3, the distribution continually shifts right since our prior had a mean of .50 while the Warriors consistenly have a winning percentage much higher. The final posterior distribution has a mean of .696, or about 57 games, which is still well below the Warriors' records over the past 3 seasons. This distibution gives them a 95% chance of winning betwee 53.8 and 60.3 games this season and the probability of them winning 73 games, like they did in the 2015-2016 season, is essentially 0. Like the Cavs, this is due to the inclusion of the first two seasons which are not representative of their more recent performance. 

### Warriors: Reduced Seasons

<img src="/post/bayesian_nba_analysis_files/figure-html/unnamed-chunk-9-1.png" width="672" style="display: block; margin: auto;" />

Just like with the Cavaliers, I will remove these outlier seasons and use a prior distribution of Beta(65,50) in order to find a more accurate posterior distribution. Again, the posterior distributions have shifted much further right than they did in the full analysis as shown in Fig. 4. The final posterior distribution has a mean of .753, predicting about 61.78 wins for the Warriors and giving them a 95% chance of winning between 58.0 and 65.3 games. The Warriors set an NBA record by winning 73 games in 2015-2016 or about 89% of their games. Even the final posterior distribution gives a probability of essentially 0 for winning this many games, indicating how historic and unpredictable that season was. The Warriors are currently on pace to win 62.73 games, which is close to the middle of the 95% interval, so removing those seasons and changing the prior seems to have been effective, although it is still too early to tell.

## Conclusions
This analysis of two NBA teams shows the benefits and drawbacks of Bayesian analysis. It is simple to implement when using conjugate distributions since the wins and losses for each season are simply added to the parameters of the Beta distribution, but there are many factors that must be taken into effect when examining the results, such as the addition/removal of players, coaches, or strategies. The use of priors must be carefully chosen to accurately reflect the true prior, especially in sports where teams and circumstances change from season to season, making the assumptions of priors less certain. However, priors provide the ability to include outside knowledge in the analysis that the data would otherwise not reflect. 

### Updates

I originally created this report as a project for a Statistical Computing class during Fall semester 2017, before the results of the NBA season were finalized. The Cavaliers ended up winning 50 games this season, very close to the mean of the posterior distribution which was 51 wins with a 95% chance of winning between 47 and 55. The Warriors only ended up winning 58 games which was at the lower edge of the 95% bayesian interval of 58-65 wins, although Steph Curry missed about 30 games this season with an MCL injury and Kevin Durant was also injured for several games with a fractured rib. Overall, the posterior distributions were fairly accurate in predicting the range of wins and show the importance of using subject expertise in a Bayesian analysis.
