---
title: "Cracking a Substitution Cypher Using MCMC methods"
summary: Finding the solution to a cypher with the Metropolis-Hastings algorithm.
date: 2020-02-20
authors: ["admin"]
tags: ["r", "algorithms", "mcmc"]
output: html_document
---



## Introduction

I recently came across a [paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.182.9688&rep=rep1&type=pdf) discussing using Markov Chain Monte Carlo methods to break substitution cyphers and wanted to implement it myself. I'll test this by encoding the first paragraph of *A Tale of Two Cities* by Charles Dickens and then use the Metropolis-Hastings algorithm to generate a solution.

## Encoding

Using the first few paragraphs of the novel should provide a long enough sample to make solving the encoding easier.


```r
text <- paste0(scan("data/cryptography/text_sample.txt", character()), collapse = " ")

text <- text %>% 
  str_remove_all(., "[;,\\.\"\'\\-_!?:()\\*\\[\\]&#@$%0-9]") 

text
```

```
## [1] "It was the best of times it was the worst of times it was the age of wisdom it was the age of foolishness it was the epoch of belief it was the epoch of incredulity it was the season of Light it was the season of Darkness it was the spring of hope it was the winter of despair we had everything before us we had nothing before us we were all going direct to Heaven we were all going direct the other way  in short the period was so far like the present period that some of its noisiest authorities insisted on its being received for good or for evil in the superlative degree of comparison only"
```

For the encoding, I'll just generate a random permutation of the letters of the alphabet and substitute the letters in the text using it.


```r
encode <- function(text){
  code <- sample(letters, 26)
  words <- str_to_lower(str_split(text, " ")[[1]]) %>% 
    str_remove_all(., "[;,\\.\"\'\\-_!?:()\\*\\[\\]&#@$%0-9]")
  new_letters <- lapply(str_split(words, ""), function(x){
    code[match(x, letters)]
  })
  new_words <- lapply(new_letters, paste0, collapse = "")
  new_text = paste0(as.vector(new_words), collapse = " ")
  list(text = new_text,
       cypher = code)
}

set.seed(101)
encoding <- encode(text)
str_sub(encoding$text, end=50)
```

```
## [1] "tp oih pzq yqhp kc ptlqh tp oih pzq okfhp kc ptlqh"
```

Decoding the text just performs the reverse operation. I also included an option that returns a vector of the integer values for use in getting values from the matrices I'll make later.


```r
decode <- function(text, cypher, as_vector = FALSE){
  # as_vector returns a vector of alphabet positions for use in matrix
  words <- str_split(str_to_lower(text), " ")[[1]]
  decoded_letters <- lapply(str_split(words, ""), function(x){
    if(as_vector){
      return(c(NA, match(x, cypher)))
    }
    letters[match(x, cypher)]
  })
  if(as_vector){
    return(unlist(decoded_letters))
  }
  
  decoded_words <- lapply(decoded_letters, paste0, collapse = "")
  new_text <- paste0(as.vector(decoded_words), collapse = " ")
  new_text
}

str_sub(decode(encoding$text, encoding$cypher), end=50)
```

```
## [1] "it was the best of times it was the worst of times"
```



## Scoring Function

In order to score possible decryptions, I need a reference text that I can compare the text to. For my reference text, I'll use *Moby Dick* by Herman Melville since it's fairly long.


```r
book <- gutenbergr::gutenberg_download(2701)$text %>% 
  paste0(., collapse = " ") %>% 
  str_remove_all(., "[;,\\.\"\'\\-_!?:()\\*\\[\\]&#@$%0-9]") %>% 
  str_to_lower()
```

Next, I'll convert the book into a vector of integer values corresponding to the letters and use this to create a matrix of frequencies for pairs of letters.


```r
codes <- match(str_split(book, "")[[1]], letters)

trans_mat <- matrix(0, nrow = 26, ncol = 26)
row.names(trans_mat) <- letters
colnames(trans_mat) <- letters

for(i in 1:26){
  # get the letters following each and ensure no zeros by adding 1:26
  following <- as.vector(table(c(codes[which(codes==i)+1], 1:26)))
  trans_mat[i, ] <- following
}
```

The scoring function for each candidate needs to measure how close to English a decoding it produces. It does this by comparing the frequency of pairs of letters in the decoding to frequencies of the same pairs as they appear in a text of actual English. Since I have the matrix of frequencies in *Moby Dick*, and combinations such as a,d should appear more frequently than combinations such as z,t, I can make a similar matrix for the result of decoding with a candidate and compare the two of them.

To do this, I'll use the scoring function from the paper mentioned earlier which takes the frequency of pairs of letters in the reference text, $ r(\beta_1, \beta_2) $, and raise it to the power of the frequency for the pairs of letters in the result, $ f(\beta_1, \beta_2) $.

$$
S(\hat{\theta})=\prod_{\beta_1,\beta_2}r(\beta_1,\beta_2)^{f(\beta_1,\beta_2)}
$$

This function will score highly when combinations with high frequencies in the reference text also have high frequencies in the encoded text. I'll take the log of this function in order to make the calculation easier.


```r
scoring <- function(text, candidate){
  test <- decode(text, candidate, TRUE)
  test_mat <- matrix(0, nrow = 26, ncol = 26)
  for(i in 1:26){
    # get the letters following each and ensure no zeros
    following <- as.vector(table(c(test[which(test==i)+1], 1:26)))
    test_mat[i, ] <- following
  }
  sum(test_mat*log(trans_mat))
}
```

At each iteration, new solutions will be generated by randomly transposing two letters in the permutation. Then this solution will be accepted with probability:

$$
r = \text{min}[1,\exp({\text{log}(S(\hat{\theta}))-\text{log}(S(\theta)}))]
$$



## Algorithm

I'll run four chains using 10,000 iterations each and make sure that the best solution overall will be returned along with records of all the scores and solutions. I'm running multiple chains to ensure that the chains find reasonable solutions.


```r
library(parallel)
library(foreach)
library(doParallel)

num_cores <- detectCores()
registerDoParallel(num_cores)

n <- 1e4

k <- 4
results <- foreach (i=1:k) %dopar% {
  start <- sample(letters, 26)
  scores <- numeric(n)
  scores[1] <- scoring(encoding$text, start)
  
  codes <- list()
  codes[[1]] <- start
  
  count <- 0
  best <- NULL
  for(i in 2:n){
    candidate <- switch_letters(codes[[i-1]])
    score <- scoring(encoding$text, candidate)
    r <- exp(score-scores[i-1])
    
    if(runif(1) < r){
      count <- count + 1
      codes[[i]] <- candidate
      scores[i] <- score
      if(is.null(best) || score > best){
        best <- score
        best_candidate <- candidate
      }
    } else {
      codes[[i]] <- codes[[i-1]]
      scores[i] <- scores[i-1]
    }
    
  }
  
  acp.rate <- count/n
  list(acp.rate = acp.rate,
       scores = scores,
       best_candidate = best_candidate,
       codes = codes)
    
}
```

<img src="/post/cryptography_files/figure-html/unnamed-chunk-3-1.png" width="480" style="display: block; margin: auto;" />

The trace plot shows that the algorithm quickly improves before reaching a good area to explore. It also shows that using multiple runs was good since one of the chains underperformed the others.



## Results


```
##           Score Acp_Rate
## 1      5805.711   0.0709
## 2      5564.466   0.0705
## 3      5805.711   0.0667
## 4      5805.711   0.0634
## Actual 5805.711       NA
```

Three of the chains found the correct solution while the fourth is unintelligible.


```
## [[1]]
## [1] "it was the best of times it was the worst of times"
## 
## [[2]]
## [1] "ha win ale vena of ahyen ha win ale wosna of ahyen"
## 
## [[3]]
## [1] "it was the best of times it was the worst of times"
## 
## [[4]]
## [1] "it was the best of times it was the worst of times"
```



```
## [1] "it was the best of times it was the worst of times it was the age of wisdom it was the age of foolishness it was the epoch of belief it was the epoch of incredulity it was the season of light it was the season of darkness it was the spring of hope it was the winter of despair we had everything before us we had nothing before us we were all going direct to heaven we were all going direct the other way  in short the period was so far like the present period that some of its noisiest authorities insisted on its being received for good or for evil in the superlative degree of comparison only"
```



Instead of trying to brute force all 26! possible permutations of the letters of the alphabet, this method was able to use probability to find the correct solution.
