---
title: "Sportscraper Package"
summary: An R package for scraping data from sports-reference.com
authors: ["admin"]
date: 2018-10-01
tags: ["r", "package", "software development"]
output: html_document
---



## Introduction

The [sportscraper](https://github.com/kmacdon/sportscraper) package is an R package I am currently developing that allows one to easily scrape data from [sports-reference.com](https://www.sports-reference.com/) for all the four major North American sports and college basketball. The package currently allows the user to easily download a player's career statistics into a data frame and I am working on adding similar functions for downloading statistics for specific teams and seasons.  

I started working on this package because many of my projects have involved using sports data, and rather than repeatedly rewrite code, I decided to create my own package by following *R Packages* by Hadley Wickam. For instructions on usage, you can follow the link above to the github repo.

## Code

The main functionality of the package comes from the `rvest` package which makes it simple to download html pages and then scrape tables from the html. Most of the code is focused on cleaning the tables and formatting them correctly. Originally I just used column indices to extract the relevant data, but I have updated using string detection to make it more robust in case sports-reference.com changes the formatting. For leagues such as the NBA and the MLB that also include advanced statistics, the `player_stats` function has an additional argument that allows the user to download that data instead of normal statistics.

## Similar Packages

The only package I have found that is similar to this is the [ballr](https://cran.r-project.org/web/packages/ballr/index.html) package which provides more options for downloading statistics, but is exclusive to the NBA. My goal is to provide a slightly simplified version of this but one that is available for all of the major sports instead of just one league.
