---
title: "Comparison of Naive KNN and K-D Tree KNN"
summary: Comparing the effectiveness of two implementations of the KNN algorithm
date: 2019-08-06
authors: ["admin"]
tags: ["r", "algorithms"]
output: html_document
---



## Introduction

The K-Nearest Neighbor algorithm is one of the simplest machine learning algorithms to understand. Starting with a labeled training set of data , you take each point in the test set and find the point in the training set that it is closest to, using that training point's label to predict the test point's label. There are modifications that can be made such as using a majority vote of the k closest points (hence the name) or weighting the votes based on distance, but fundamentally it's all the same. It's an intuitive algorithm and one that is relatively simple to implement, but how it's implemented can make a huge difference in its effectiveness. I'll explore this by comparing two different implementations, one using a naive approach and one using a data structure called a k-D tree which will I will explain later on.

## Naive Approach

The simplest way to find the point in the training set closest to one in the test set is simply to compare the test point to every single point in the training set and decide which one is closest. It might already be apparent that the speed this algorithm runs is heavily dependent on how many points you have in each set. If you have n training points and m testing points then you have to make n*m comparisons, so the growth is exponential as your sets grow. 

### Implemenation

The first part of creating a KNN algorithm is to decide which distance metric to use for the comparisons. For this post, I'll use the most common distance, Euclidean distance. This R code simply takes two vectors and calculates that distance.


```r
euclidean_dist <- function(a, b){
  sqrt(sum((a-b)^2))
}
```

Now that we have the distance metric, we can move on to the actual algorithm. As said before, this code just loops through every point in the testing set, and for each of those points, it loops through every point in the training set and finds which point has the smallest distance, then takes that point's classification as the testing point's classification. 


```r
nearest_neighbor_naive <- function(training, classes, testing){
  classification <- character(nrow(testing))
  
  for(i in 1:nrow(testing)){
    min <- NULL
    neighbor <- NULL
    
    for(j in 1:nrow(training)){
      dist <- euclidean_dist(testing[i, ], training[j, ])
      if(is.null(min) || dist < min){
        min <- dist
        neighbor <- j
      }
    }
    
    classification[i] <- classes[neighbor]
    
  }
  
  classification
}
```

## K-D Tree Approach

A k-D tree is a way of storing data that makes it much more efficient to search through the points and find the closest neighbor. The way a k-D tree works is, as the name might suggest, by storing the data in a tree. Starting with the root node, you pick the first column in the data and find the median value. Then you create two child branches, one with all the points whose value in the first column are less than the median value, and the other with all the points whose value in the first column are greater than the median value. The median point is stored at that node. Then for each of the child nodes you repeat the process using the second column instead. You keep repeating this process, cycling through the columns, until every point is stored at a node. For more information, check out the [Wikipedia](https://en.wikipedia.org/wiki/K-d_tree) post on the subject since it gives a very easy to understand description.

### Implementation

### Creating a K-D Tree

To start off, I'll run through the process of actually creating the tree itself and break it up into pieces.

The inputs to the tree are the training data set, the class labels for each point in the data set, and the current depth of the tree. This last argument is important since the tree is built recursively and we need to be able to keep track of the depth.


```r
kd_tree <- function(data, classes, depth = 1)
```

Since this tree will likely have many layers of branches, I need to make sure the `depth` variable stays within the bounds of the number of columns, which can be achieved using the modulus operator


```r
  depth <- depth %% ncol(data)
  
  if(!depth){
    # In case depth is 0 
    depth <- ncol(data)
  }
```

I reset the depth if it is 0 since R indices start at 1, but this wouldn't be an issue in other languages.

Since this is a recursive algorithm, I need to focus on the base case, when there is only one (or zero) data point passed to the function. Once this case is reached, I create a node, recording the point, it's median value, the column of that median value, and set the right and left nodes to `NULL` since there are no more points to create child brances with.


```r
  # If only one point, make node
  if(nrow(data) == 1){
    
    # Check to see if data is empty
    if(is.na(data[depth])){
      return(NULL)
    }
    
    # Else return node
    return(list(
      column = depth,
      value = data[depth],
      point = as.vector(data),
      class = classes,
      left = NULL,
      right = NULL
    ))
  }
```

Now that the base case has been taken care of, I can focus on the case when there are multiple points left. In this case, you use the current column and find the median value, then split the data up into left and right data sets. You'll notice I don't use R's `median` function even though that would be simpler. This is because I need to know the index of the median point so that I can store it in the node. There would also be an issue if I only had two points with values say 1 and 2 since the median would then be 1.5 and not helpful. 

Once the data is split, I create a node just like above, but this time I create a left and right child branch by calling the `kd_tree` function again on the subsets of the data, increasing the depth for each by 1 as I do so. When I divide the data, I use the `drop=FALSE` option to make sure the data stays as a matrix and doesn't turn into a vector if there is only one point.


```r
  # Use sorting and integer division to find median
  # Need to ensure median corresponds to actual data point
  
  middle <- nrow(data) %/% 2
  data_order <- order(data[, depth])
  
  # Split data excluding median point
  left_data <- data[data_order[1:(middle-1)], , drop=FALSE]
  right_data <- data[data_order[(middle+1):nrow(data)], , drop=FALSE]
  middle_point <- data[data_order[middle], ]
  
  return(list(
    column = depth,
    value = middle_point[depth],
    point = middle_point,
    class = classes[data_order[middle]],
    left = kd_tree(left_data, classes[data_order[1:(middle-1)]], depth + 1),
    right = kd_tree(right_data, classes[data_order[(middle+1):nrow(data)]], depth + 1)
  ))
  
}
```

Now that the k-D tree is taken care of, I can move on to how to use it to find the nearest neighbor of a new point. 

#### Finding the Nearest Neighbor

The inputs to this function are the k-D tree created from the previous function, the point to find a neighbor for, and the current closest neighbor, both its class and distance from the testing point. To start off, both these values will be `NULL`.


```r
find_neighbor <- function(tree, point, best = list(class = NULL, dist = NULL))
```

Since this is another recursive algorithm, I need to take care of the base case, which is when the algorithm gets to a branch that doesn't exist. In this case, it simply goes back up one branch.


```r
  # Go back up one if null node
  if(is.null(tree)){
    return(best)
  }
```

If the branch does exist, then I use the column of the point at the node and the median value to decide whether the algorithm should go down the left branch or the right branch, passing down the current closest neighbor as it goes.


```r
  if(point[tree$column] < tree$value){
    best <- find_neighbor(tree$left, point, best)
  } else {
    best <- find_neighbor(tree$right, point, best)
  }
```

Once these functions have returned a value, I have to compare that value to the current node and see if it is closer than the current closest point.


```r
  # Compare current node to best or set it as best if none set
  if(is.null(best$dist) || euclidean_dist(tree$point, point) < best$dist){
    best <- list(class = tree$class,
                 dist = euclidean_dist(tree$point, point))
  }
```

After checking the current node, I then have to decide if I need to head down the other branch the algorithm didn't take before to see if there is a closer point on that side. The Wikipedia article explains the theory behind this, but if the distance between the current node's value and the point's value is less than the distance between the point and it's current closest neighbor, there is a chance that a better candidate exists on that side.


```r
  # Check to see if the other branch needs to be checked 
  
  if(abs(tree$value - point[tree$column]) < best$dist){
    if(point[tree$column] > tree$value){
      best <- find_neighbor(tree$left, point, best)
    } else {
      best <- find_neighbor(tree$right, point, best)
    }
  }
```

Once that is done, I can return the current best point and recursion will take care of the rest.

Now that the algorithm is complete, I can combine the two functions together into the full classifier and test it with the same data used before.


```r
nearest_neighbor_tree <- function(training, classes, testing){
  neighbors <- numeric(nrow(testing))
  tree <- kd_tree(training, classes)
  for(i in 1:nrow(testing)){
      
    neighbors[i] <- find_neighbor(tree, testing[i, ])$class
    
  }
  
  neighbors
}
```

## Evaluation

In order to evaluate the algorithms, I'll generate a a matrix of random uniform numbers with two columns and classify the points based on their sum. I'll use differing numbers of points to compare the speeds as the data grows.


```r
n <- c(1e2, 1e3, 1e4, 1e5)
naive_results <- list()
kd_results <- list()

for(i in 1:length(n)){
  set.seed(101)
  n_train <- n[i]
  n_test <- .1*n_train
  
  train <- matrix(runif(n_train*2), nrow = n_train)
  class <- ifelse(train[, 1] + train[, 2] > .7, "a", "b")
  test <- matrix(runif(n_test*2), nrow = n_test)
  truth <- ifelse(test[, 1] + test[, 2] > .7, "a", "b")
  
  naive_results[[i]] <- microbenchmark::microbenchmark(nearest_neighbor_naive(train, class, test), times = 1)
  kd_results[[i]] <- microbenchmark::microbenchmark(nearest_neighbor_tree(train, class, test), times = 1)
    
}
```
<img src="/post/kdtree_comparison_files/figure-html/plot_evals-1.png" width="672" />

The performances are fairly similar for the data sets with fewer observations, but there is a huge performance penalty for the naive method once there are 100,000 points, but the tree method is barely affected. With that many points, the naive method takes about 37 minutes while the tree method takes just 3.9 seconds.



## Conclusion

Hopefully this example has shown that the way data is stored can be extremely important in how an algorithm performs. The accuracy is the same between them, so there is absolutely no reason to choose the naive implemenatation. The k-D tree approach might take more time to develop than the naive approach, but the speed gains are obviously worth it. 

### Addition

I ended up also coding the algorithm in C++ using the RCpp package just to see how much faster it would be. 


```r
n <- c(1e2, 1e3, 1e4, 1e5)
cpp_results <- list()

for(i in 1:length(n)){
  set.seed(101)
  n_train <- n[i]
  n_test <- .1*n_train
  
  train <- matrix(runif(n_train*2), nrow = n_train)
  class <- ifelse(train[, 1] + train[, 2] > .7, 1, 0)
  test <- matrix(runif(n_test*2), nrow = n_test)
  truth <- ifelse(test[, 1] + test[, 2] > .7, 1, 0)
  
  cpp_results[[i]] <- microbenchmark::microbenchmark(KDcpp::nn_classification_cpp(train, test, class))
}
```

<img src="/post/kdtree_comparison_files/figure-html/plot_cpp-1.png" width="672" />

The times for both algorithms are pretty reasonable, with the R version taking just 4 seconds for 100,000 rows, but the C++ version was still about 4 times as fast. It took me significantly longer to code than the R version for just a small improvement, but that difference could be meaningful in certain scenarios. I'll also see how the algorithms compare when using 50 columns instead of 2.


```r
n <- c(1e2, 1e3, 1e4)
ncol <- 50
cpp_results <- list()
kd_results <- list()

for(i in 1:length(n)){
  set.seed(101)
  n_train <- n[i]
  n_test <- .1*n_train
  
  train <- matrix(runif(n_train*ncol), nrow = n_train)
  class <- ifelse(train[, 1] + train[, 2] + train[, 3] + train[, 40] > 2, 1, 0)
  test <- matrix(runif(n_test*ncol), nrow = n_test)
  truth <- ifelse(test[, 1] + test[, 2] + test[, 3] + test[, 40] > 2, 1, 0)
  
  kd_results_cols[[i]] <- microbenchmark::microbenchmark(nearest_neighbor_tree(train, class, test), times=10)
  cpp_results_cols[[i]] <- microbenchmark::microbenchmark(KDcpp::nn_classification_cpp(train, test, class), times=10)
}
```

<img src="/post/kdtree_comparison_files/figure-html/plot_col-1.png" width="672" />

Upping the columns to 50 cause a dramatic decrease in the speed of the R algorithm while the C++ one barely changed at all. Even though writing the algorithm in C++ took much longer, it clearly was worth it since it's 80 times faster than the R version for the most extreme case shown, taking less than a second.
