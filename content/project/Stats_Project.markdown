---
title: Analysis of NBA Box Score Data
summary: Inferential analysis of NBA box score data using Logistic Regression
date: 2017-11-01
authors: ["admin"]
tags: ["r", "analysis"]
output: html_document
---


## Introduction
  For this project, I collected and analyzed the box score data of NBA teams in order to determine which statistics were most relevant in determining the winner of a game and find if there are any variables that do not significantly affect the outcome. I acquired this data from [Basketball-Reference.com](https://www.basketball-reference.com) by scraping the box scores of every game played by every team for the 2016-2017 NBA season and then limited my analysis to each team's home games in order to avoid double counting games and remove the effects of home court advantage. The data set contains a binary response variable, W, with 1 being a win for the home team, as well as variables for points, field goal percentage, 3 point field goal percentage, offensive rebounds, total rebounds, assists, steals, and turnovers for both the home and away team, for a total of 16 predictive variables. 

## Purposeful Selection of Covariates

In order to create our model, we will go through a series of steps designed to remove insignificant variables while looking for any special relationships between the variables such as non-linearity or statistical interaction between predictors. The steps I follow come from the method of variable selection as described in *Applied Logistic Regression* (Hosmer 1989).

##### Step 1

For this step, I will fit every variable to the response in a univariable model to see if there are any that are not significant predictors.


```r
# Step 1 of PSOC
# Calculate statistics for all the univariable models and remove variables that 
# are not significant

n <- length(2:dim(HomeBoxScores)[2])
coeff <- numeric(n)
p.values <- numeric(n)
std.err <- numeric(n)
G <- numeric(n)

#Fit all univariable models and record the statistics
for(i in 2:dim(HomeBoxScores)[2]){
  mod <- glm(HomeBoxScores$W ~ HomeBoxScores[, i], family = binomial)
  coeff[i-1] <- mod$coefficient[2]
  p.values[i-1] <- coef(summary(mod))[2, 4]
  std.err[i-1] <- coef(summary(mod))[2, 2]
  G[i-1] <- mod$null.deviance - mod$deviance
}

OR <- exp(coeff)

results <- data.frame(cbind(coeff, p.values, std.err, OR, G), 
                      row.names = names(HomeBoxScores)[2:dim(HomeBoxScores)[2]])
knitr::kable(round(results,3))
```



|      |   coeff| p.values| std.err|           OR|       G|
|:-----|-------:|--------:|-------:|------------:|-------:|
|PTS   |   0.091|     0.00|   0.007| 1.095000e+00| 252.132|
|O_PTS |  -0.105|     0.00|   0.007| 9.000000e-01| 322.809|
|FG.   |  20.185|     0.00|   1.472| 5.836586e+08| 250.969|
|X3P.  |   8.021|     0.00|   0.733| 3.043748e+03| 142.888|
|TRB   |   0.098|     0.00|   0.010| 1.103000e+00| 102.615|
|AST   |   0.139|     0.00|   0.013| 1.149000e+00| 131.728|
|STL   |   0.132|     0.00|   0.021| 1.141000e+00|  40.047|
|TOV   |  -0.039|     0.01|   0.015| 9.620000e-01|   6.629|
|O_FG. | -22.682|     0.00|   1.536| 0.000000e+00| 304.831|
|O_3P. |  -9.376|     0.00|   0.774| 0.000000e+00| 181.684|
|O_TRB |  -0.073|     0.00|   0.010| 9.300000e-01|  59.152|
|O_AST |  -0.121|     0.00|   0.013| 8.860000e-01| 101.588|
|O_STL |  -0.074|     0.00|   0.020| 9.290000e-01|  13.665|
|O_TOV |   0.084|     0.00|   0.016| 1.087000e+00|  28.661|

We can see that all variables are significant at the 1% level when fit as univariable models, so I will fit a model with every variable as a predictor.

##### Step 2

```r
# Step 2
# Fit model with all variables that were significant from step 1

mod.2 <- glm(W ~ . - PTS - O_PTS , data = HomeBoxScores, family = binomial)
knitr::kable(round(coef(summary(mod.2)), 3))
```



|            | Estimate| Std. Error| z value| Pr(>&#124;z&#124;)|
|:-----------|--------:|----------:|-------:|------------------:|
|(Intercept) |    3.797|      3.444|   1.102|              0.270|
|FG.         |   38.316|      4.052|   9.457|              0.000|
|X3P.        |    9.485|      1.523|   6.227|              0.000|
|TRB         |    0.218|      0.026|   8.294|              0.000|
|AST         |    0.098|      0.029|   3.392|              0.001|
|STL         |    0.052|      0.061|   0.844|              0.399|
|TOV         |   -0.467|      0.056|  -8.392|              0.000|
|O_FG.       |  -47.028|      4.424| -10.630|              0.000|
|O_3P.       |  -11.428|      1.607|  -7.112|              0.000|
|O_TRB       |   -0.234|      0.027|  -8.682|              0.000|
|O_AST       |   -0.042|      0.027|  -1.568|              0.117|
|O_STL       |   -0.032|      0.060|  -0.525|              0.599|
|O_TOV       |    0.483|      0.057|   8.436|              0.000|

The algorithm cannot converge for our model if both PTS and O_PTS are used as predictors, since just those 2 variables can perfectly predict the winner of a game. To avoid this problem, I removed both from the model; this will also allow us to focus on the more interesting variables since we already know how important PTS and O_PTS are to winning. We can see that STL, O_AST, and O_STL are not significant at the 5% level in this model, so I will remove them and test for their effects on the other variables in the next step. 

##### Step 3

```r
# Step 3
# Check for statistical adjustment with variables that were removed in step 2

comp.df <- HomeBoxScores %>% 
  dplyr::select(-PTS, -O_PTS, -O_AST, -STL, -O_STL)

n <- length(comp.df) - 1
n.coeff <- numeric(n)
v.names <- numeric(n)
a.name <- numeric(n)
change <- numeric(n)

mod <- glm(W ~ ., data = comp.df, family = binomial)
coeff <- mod$coefficients[-1]
vars <- c(8, 13:14) # Column index of variables that were not significant.

# Fit new models with each of the removed variables and compare the coefficients 
# to those of the model without the variables
for (i in 1:length(vars)) {
  mod <- glm(W ~ . + HomeBoxScores[, vars[i]] , data = comp.df, family = binomial)
  n.coeff <- mod$coefficients[2:dim(comp.df)[2]]
  v.names <- names(comp.df)[-1]
  a.name <- rep(names(HomeBoxScores)[vars[i]], length(n.coeff))
  change <- as.numeric((coeff - n.coeff)/n.coeff)
  if (i == 1) {
    results.2 <- data.frame(Variable = v.names, Adjuster = a.name, Change = change)
  } else {
    results.2 <- rbind(results.2, data.frame(Variable = v.names, Adjuster = a.name, Change = change))
  }
}
# Did any coefficients change at least 18%
any(abs(results.2$Change) > .18)
```

```
## [1] FALSE
```

None of the removed variables significantly changed the coefficients of the other predictors when tested with the largest change being just 9.5%. There is no need to add any of the variables back to the model since they do not adjust any of the other predictors.

##### Step 4


```r
# Step 4
HomeBoxScores <- comp.df
mod.prelim <- glm(W ~ ., data = HomeBoxScores, family = binomial)
```

Since no variables were removed in Step 1, there are no variables that need to be added to the model to see if they are significant in a multivariable logistic regression. Our preliminary model is:

W ~ FG., X3P., TRB, AST, TOV, O_FG., O_3P., O_TRB, O_TOV

##### Step 5

In order to determine any non-linear relationships between the variables and the logit, I will construct Lowess plots for all of the variables in our model.


```r
# Step 5
par(mfrow = c(3,3))

for(i in 2:dim(HomeBoxScores)[2]){
  logitloess(HomeBoxScores[, i], HomeBoxScores$W, .7)
  title(paste("Lowess Plot for",names(HomeBoxScores)[i],sep = ' '))
}
```

<img src="/project/Stats_Project_files/figure-html/unnamed-chunk-5-1.png" width="864" style="display: block; margin: auto;" />

The Lowess plots appear to show obvious non-linearity with the variable TOV as well as potential non-linear relationships between W and FG%, O_3P%, and O_TRB. so I will perform analysis using fractional polynomials in order to determine if any transformation of these variables is necessary to better fit the relationship.


```r
# Fractional Polynomails
fit1 <- mfp(W ~ fp(TOV),family=binomial, data=HomeBoxScores,verbose=T)
pchisq(1661.89 - 1659.5, 2, lower.tail = F)
pchisq(1663.85 - 1659.5, 3, lower.tail = F)
pchisq(1670.478 - 1659.5, 4, lower.tail = F)

# FG%
fit2 <- mfp(W ~ fp(HomeBoxScores[,2]),family=binomial,data=HomeBoxScores,verbose=T)
pchisq(1417.429 - 1417.16, 2, lower.tail = F)
pchisq(1419.51 - 1417.16, 3, lower.tail = F)
pchisq(1670.478 - 1417.16, 4, lower.tail = F)

# O_TRB
fit3 <- mfp(W ~ fp(HomeBoxScores[, 9]), family=binomial,data=HomeBoxScores,verbose=T)
pchisq(1607.829 - 1607.529, 2, lower.tail = F)
pchisq(1611.326 - 1607.529, 3, lower.tail = F)
pchisq(1670.478 - 1607.529, 4, lower.tail = F)

# O_3P%
fit4 <- mfp(W ~ fp(HomeBoxScores[, 8]), family=binomial,data=HomeBoxScores,verbose=T)
pchisq(1487.726 - 1486.613, 2, lower.tail = F)
pchisq(1488.794 - 1486.613, 3, lower.tail = F)
pchisq(1670.478 - 1486.613, 4, lower.tail = F)
```

Despite what the Lowess plots show, the fractional polynomials that were fit to these variables all indicate that they should remain linear since none of the models with fractional polynomials were significantly better than the linear models. The slight non-linearity that appears in the plots for the other variables must not be enough to warrant using fractional polynomials, so we will continue using our model with untransformed variables.

##### Step 6

In order to see if there are any significant interactions between variables, I will fit models with all possible 2 variable interaction terms.


```r
# Step 6
pval <- numeric(sum(1:8))
names <- numeric(sum(1:8))
count <- 0
# For loop fits all possible 2 variable interaction terms and records
# pvalues
for(i in 2:(dim(HomeBoxScores)[2]-1)){
  for(j in (i+1):dim(HomeBoxScores)[2]){
    count <- count + 1
    mod <- glm(W ~ . + HomeBoxScores[, i]: HomeBoxScores[, j], data = HomeBoxScores, 
               family = binomial)
    pval[count] <- coef(summary(mod))[11, 4]
    names[count] <- paste(names(HomeBoxScores)[i],":",names(HomeBoxScores)[j], sep="")
  }
}
results.6 <- data.frame(PVal = pval, Interaction = names) %>% 
  filter(PVal < .05)
results.6
```

```
##          PVal Interaction
## 1 0.004820983   AST:O_FG.
```

The only significant interaction term is between AST and O_FG%, although it is barely significant. Adding the interaction term to the model caused the coefficient for AST to become negative which is counterintuitive. According to this model every extra assist lowers the log-odds of winning by .589, so each assist is actually more harmful than a turnover since TOV has a coefficient of -.496. This does not make sense since assists are beneficiary because they directly lead to points, so the interaction term will not be added to the model.




```r
mod.final <- glm(W ~ . , data = HomeBoxScores, family = binomial)
```

## Diagnostics


```r
hoslem.test(mod.final$fitted.values, W, g = 10)
```

```
## 
## 	Hosmer and Lemeshow goodness of fit (GOF) test
## 
## data:  mod.final$fitted.values, W
## X-squared = 1.6878e-19, df = 8, p-value = 1
```

The Hoslem-Lemeshow test reported a p-value of 1, so we will not reject the null hypothese and conclude that our model is a good fit for the data. This is likely because our model is overfit, but our main goal is to analyze the significance it gives to each predictor so we will continue with this model.


```r
# Cooks Distance
cooks <- cooks.distance(mod.final)
cooks[abs(cooks)>1]
```

```
## named numeric(0)
```

```r
# 5-Folds Cross Validation
l <- c(1,247,492,738,983)
u <- c(246,491,737,983,1229)
err.rate <- numeric(5)
for(i in 1:5){
  mod <- glm(W ~ ., data = HomeBoxScores[-(l[i]:u[i]), ], family = binomial)
  y <- plogis(predict(mod, HomeBoxScores[l[i]:u[i], ]))
  yhat <- round(y)
  err.rate[i] <- length(which(yhat != W[l[i]:u[i]]))/length(W)
}
err.rate
```

```
## [1] 0.01544715 0.02032520 0.02032520 0.01951220 0.01626016
```

None of the Cook's distances for the observations was greater than one, so we do not have any influential observations in our data that we need to remove. In order to validate our model, we performed a 5-folds cross validation test. None of the error rates for the 5 tests were greater than about 2%, so our model performed very well and there is no need to change it.

## Conclusions


```r
values <- coef(summary(mod.final))[,1]
lower <- values - 1.96*sqrt(coef(summary(mod.final))[,2])
upper <- values + 1.96*sqrt(coef(summary(mod.final))[,2])
knitr::kable(round(cbind(values,lower,upper), 4))
```



|            |   values|    lower|    upper|
|:-----------|--------:|--------:|--------:|
|(Intercept) |   4.7553|   1.1617|   8.3489|
|FG.         |  37.9934|  34.0690|  41.9177|
|X3P.        |   9.3810|   6.9688|  11.7933|
|TRB         |   0.2113|  -0.1033|   0.5260|
|AST         |   0.0888|  -0.2390|   0.4166|
|TOV         |  -0.4901|  -0.8912|  -0.0890|
|O_FG.       | -48.8926| -52.9535| -44.8317|
|O_3P.       | -11.6024| -14.0859|  -9.1189|
|O_TRB       |  -0.2391|  -0.5583|   0.0801|
|O_TOV       |   0.5131|   0.0953|   0.9310|

The final model produced was very good at predicting wins or losses, but that should not be a surprise since the box score data is collected after the game. Therefore, we already know how each team performed in the game before predicting which team won. Comparing the coefficients of our model, each rebound was more beneficial to winning than assists since $ \beta\_{TRB} = .211 $ and $ \beta\_{AST} = .089 $) and the p-value for AST was much larger than that of TRB, so our model found each rebound to be more important to winning than each assist. It is a surprising result since rebounds are more common and do not directly result in points unlike assists.

Interestingly, the magnitudes of the coefficients for all the defensive variables were greater than the those of their respective offensive variables, so our model indicates that playing well defensively is more valuable than playing well offensively, although the magnitudes were not vastly different. However, our model also found that steals were not significant predictors, but this may be due to their relative rarity in a basketball game and their possible correlation with turnovers. It would have been interesting to include advanced statistics such as effective field goal percentage or free throw rate to see how superior they are to traditional box score statistics. 

A good future project based off of this would be using spatial statistics in order to analyze games on a more in-depth level. Factoring in fatigue such as accounting for how many games have been played in the last 5 days might also have been a good addition. Overall, this model helped reveal some of the important factors in winning NBA games, although there are many possible predictors that could have been used. I chose the basic box scores for simplicity, but in the future, exploring these advanced statistics would make for a more complete analysis.
