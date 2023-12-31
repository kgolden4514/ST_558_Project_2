Project 2
================
Kristina Golden and Demetrios Samaras
2023-07-02

# Business

## Required Packages

``` r
library(tidyverse)
library(knitr)
library(GGally)
library(corrplot)
library(qwraps2)
library(vtable)
library(psych)
library(ggplot2)
library(cowplot)
library(caret)
library(gbm)
library(randomForest)
library(tree)
library(class)
library(bst)
library(reshape)
library(reshape2)
library(corrr)
library(ggcorrplot)
library(FactoMineR)
library(factoextra)
library(data.table)
```

## Introduction

In this report we will be looking at the Business data channel of the
online news popularity data set. This data set looks at a wide range of
variables from 39644 different news articles. The response variable that
we will be focusing on is **shares**. The purpose of this analysis is to
try to predict how many shares a Business article will get based on the
values of those other variables. We will be modeling shares using two
different linear regression models and two ensemble tree based models.

## Read in the Data

``` r
#setwd("C:/Documents/Github/ST_558_Project_2")
setwd("C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2")
 

online <- read.csv("OnlineNewsPopularity.csv")
colnames(online) <- c('url', 'days', 'n.Title', 'n.Content', 'Rate.Unique', 
                      'Rate.Nonstop', 'Rate.Unique.Nonstop', 'n.Links', 
                      'n.Other', 'n.Images', 'n.Videos',
                      'Avg.Words', 'n.Key', 'Lifestyle', 'Entertainment',
                      'Business', 'Social.Media', 'Tech', 'World', 'Min.Worst.Key',
                      'Max.Worst.Key', 'Avg.Worst.Key', 'Min.Best.Key', 
                      'Max.Best.Key', 'Avg.Best.Key', 'Avg.Min.Key', 'Avg.Max.Key',
                      'Avg.Avg.Key', 'Min.Ref', 'Max.Ref', 'Avg.Ref', 'Mon', 
                      'Tues', 'Wed', 'Thurs', 'Fri', 'Sat', 'Sun', 'Weekend',
                      'LDA_00', 'LDA_01', 'LDA_02', 'LDA_03', 'LDA_04', 
                      'Global.Subj', 'Global.Pol', 'Global.Pos.Rate',
                      'Global.Neg.Rate', 'Rate.Pos', 'Rate.Neg', 'Avg.Pos.Pol',
                      'Min.Pos.Pol', 'Max.Pos.Pol', 'Avg.Neg.Pol', 'Min.Neg.Pol',
                      'Max.Neg.Pol', 'Title.Subj', 'Title.Pol', 'Abs.Subj',
                      'Abs.Pol', 'shares')
#Dropped url and timedelta because they are non-predictive. 
online <- online[ , c(3:61)]
```

## Write Functions

``` r
summary_table <- function(data_input) {
    min <- min(data_input$shares)
    q1 <- quantile(data_input$shares, 0.25)
    med <- median(data_input$shares)
    q3 <- quantile(data_input$shares, 0.75)
    max <- max(data_input$shares)
    mean1 <- mean(data_input$shares)
    sd1 <- sd(data_input$shares)
    data <- matrix(c(min, q1, med, q3, max, mean1, sd1), 
                   ncol=1)
    rownames(data) <- c("Minimum", "Q1", "Median", "Q3",
                           "Maximum", "Mean", "SD")
    colnames(data) <- c('Shares')
    data <- as.table(data)
    data
}
```

``` r
#Create correlation table and graph for a training dataset
correlation_table <- function(data_input) {
  #drop binary variables
  correlations <- cor(subset(data_input, select = c(2:4, 6:24,
                                                    33:50)))
  kable(correlations, caption = 'Correlations Lifestyle')
}
```

``` r
# Create correlation graph
correlation_graph <- function(data_input,sig=0.5){
  corr <- cor(subset(data_input, select = c(2:4, 6:24, 33:50)))
  corr[lower.tri(corr, diag = TRUE)] <- NA
  corr <- melt(corr, na.rm = TRUE)
  corr <- subset(corr, abs(value) > 0.5)
  corr[order(-abs(corr$value)),]
  print(corr)
  mtx_corr <- reshape2::acast(corr, Var1~Var2, value.var="value")
  corrplot(mtx_corr, is.corr=FALSE, tl.col="black", na.label=" ")
}
```

## Business EDA

### Business

``` r
## filters rows based on when parameter is 1 
data_channel <-  online %>% filter( !!rlang::sym(params$DataChannel) == 1)

## Drop the data_channel_is columns 
data_channel <- data_channel[ , -c(12:17)]

## reorder to put shares first 
data_channel <- data_channel[ , c(53, 1:52)]
```

``` r
set.seed(5432)

# Split the data into a training and test set (70/30 split)
# indices

train <- sample(1:nrow(data_channel), size = nrow(data_channel)*.70)
test <- setdiff(1:nrow(data_channel), train)

# training and testing subsets
data_channel_train <- data_channel[train, ]
data_channel_test <- data_channel[test, ]
```

## Business Summarizations

``` r
#Shares table for data_channel_train
summary_table(data_channel_train)
```

    ##            Shares
    ## Minimum      1.00
    ## Q1         957.00
    ## Median    1400.00
    ## Q3        2500.00
    ## Maximum 310800.00
    ## Mean      2889.68
    ## SD        9916.94

The above table displays the Business 5-number summary for the shares.
It also includes the mean and standard deviation. Because the mean is
greater than the median, we suspect that the Business shares
distribution is right skewed.

``` r
#Correlation table for lifestyle_train
correlation_table(data_channel_train)
```

<table>
<caption>
Correlations Lifestyle
</caption>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:right;">
n.Title
</th>
<th style="text-align:right;">
n.Content
</th>
<th style="text-align:right;">
Rate.Unique
</th>
<th style="text-align:right;">
Rate.Unique.Nonstop
</th>
<th style="text-align:right;">
n.Links
</th>
<th style="text-align:right;">
n.Other
</th>
<th style="text-align:right;">
n.Images
</th>
<th style="text-align:right;">
n.Videos
</th>
<th style="text-align:right;">
Avg.Words
</th>
<th style="text-align:right;">
n.Key
</th>
<th style="text-align:right;">
Min.Worst.Key
</th>
<th style="text-align:right;">
Max.Worst.Key
</th>
<th style="text-align:right;">
Avg.Worst.Key
</th>
<th style="text-align:right;">
Min.Best.Key
</th>
<th style="text-align:right;">
Max.Best.Key
</th>
<th style="text-align:right;">
Avg.Best.Key
</th>
<th style="text-align:right;">
Avg.Min.Key
</th>
<th style="text-align:right;">
Avg.Max.Key
</th>
<th style="text-align:right;">
Avg.Avg.Key
</th>
<th style="text-align:right;">
Min.Ref
</th>
<th style="text-align:right;">
Max.Ref
</th>
<th style="text-align:right;">
Avg.Ref
</th>
<th style="text-align:right;">
LDA_00
</th>
<th style="text-align:right;">
LDA_01
</th>
<th style="text-align:right;">
LDA_02
</th>
<th style="text-align:right;">
LDA_03
</th>
<th style="text-align:right;">
LDA_04
</th>
<th style="text-align:right;">
Global.Subj
</th>
<th style="text-align:right;">
Global.Pol
</th>
<th style="text-align:right;">
Global.Pos.Rate
</th>
<th style="text-align:right;">
Global.Neg.Rate
</th>
<th style="text-align:right;">
Rate.Pos
</th>
<th style="text-align:right;">
Rate.Neg
</th>
<th style="text-align:right;">
Avg.Pos.Pol
</th>
<th style="text-align:right;">
Min.Pos.Pol
</th>
<th style="text-align:right;">
Max.Pos.Pol
</th>
<th style="text-align:right;">
Avg.Neg.Pol
</th>
<th style="text-align:right;">
Min.Neg.Pol
</th>
<th style="text-align:right;">
Max.Neg.Pol
</th>
<th style="text-align:right;">
Title.Subj
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
n.Title
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.002403
</td>
<td style="text-align:right;">
0.003776
</td>
<td style="text-align:right;">
0.009661
</td>
<td style="text-align:right;">
-0.058927
</td>
<td style="text-align:right;">
-0.023967
</td>
<td style="text-align:right;">
-0.023733
</td>
<td style="text-align:right;">
0.042989
</td>
<td style="text-align:right;">
-0.088000
</td>
<td style="text-align:right;">
-0.002771
</td>
<td style="text-align:right;">
-0.045687
</td>
<td style="text-align:right;">
-0.024068
</td>
<td style="text-align:right;">
-0.057209
</td>
<td style="text-align:right;">
0.035840
</td>
<td style="text-align:right;">
0.055915
</td>
<td style="text-align:right;">
0.056975
</td>
<td style="text-align:right;">
0.019985
</td>
<td style="text-align:right;">
0.028279
</td>
<td style="text-align:right;">
0.035345
</td>
<td style="text-align:right;">
-0.006686
</td>
<td style="text-align:right;">
0.001985
</td>
<td style="text-align:right;">
0.002322
</td>
<td style="text-align:right;">
-0.071265
</td>
<td style="text-align:right;">
0.002299
</td>
<td style="text-align:right;">
0.050126
</td>
<td style="text-align:right;">
0.028309
</td>
<td style="text-align:right;">
0.040678
</td>
<td style="text-align:right;">
-0.015302
</td>
<td style="text-align:right;">
-0.038423
</td>
<td style="text-align:right;">
-0.018033
</td>
<td style="text-align:right;">
0.028051
</td>
<td style="text-align:right;">
-0.027998
</td>
<td style="text-align:right;">
0.029181
</td>
<td style="text-align:right;">
-0.026867
</td>
<td style="text-align:right;">
-0.003900
</td>
<td style="text-align:right;">
-0.013722
</td>
<td style="text-align:right;">
-0.017634
</td>
<td style="text-align:right;">
-0.005758
</td>
<td style="text-align:right;">
-0.003502
</td>
<td style="text-align:right;">
0.146469
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Content
</td>
<td style="text-align:right;">
-0.002403
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.721974
</td>
<td style="text-align:right;">
-0.557612
</td>
<td style="text-align:right;">
0.578959
</td>
<td style="text-align:right;">
0.148974
</td>
<td style="text-align:right;">
0.227858
</td>
<td style="text-align:right;">
0.160154
</td>
<td style="text-align:right;">
0.026064
</td>
<td style="text-align:right;">
0.167237
</td>
<td style="text-align:right;">
-0.065708
</td>
<td style="text-align:right;">
0.061507
</td>
<td style="text-align:right;">
0.062479
</td>
<td style="text-align:right;">
-0.050350
</td>
<td style="text-align:right;">
0.077450
</td>
<td style="text-align:right;">
-0.077576
</td>
<td style="text-align:right;">
0.023457
</td>
<td style="text-align:right;">
0.012473
</td>
<td style="text-align:right;">
0.039085
</td>
<td style="text-align:right;">
-0.024377
</td>
<td style="text-align:right;">
0.002683
</td>
<td style="text-align:right;">
-0.021404
</td>
<td style="text-align:right;">
0.152093
</td>
<td style="text-align:right;">
-0.069704
</td>
<td style="text-align:right;">
-0.004284
</td>
<td style="text-align:right;">
-0.065903
</td>
<td style="text-align:right;">
-0.112971
</td>
<td style="text-align:right;">
0.174223
</td>
<td style="text-align:right;">
0.090778
</td>
<td style="text-align:right;">
0.178952
</td>
<td style="text-align:right;">
0.090854
</td>
<td style="text-align:right;">
0.057098
</td>
<td style="text-align:right;">
-0.023737
</td>
<td style="text-align:right;">
0.113263
</td>
<td style="text-align:right;">
-0.332958
</td>
<td style="text-align:right;">
0.463758
</td>
<td style="text-align:right;">
-0.143772
</td>
<td style="text-align:right;">
-0.489611
</td>
<td style="text-align:right;">
0.276593
</td>
<td style="text-align:right;">
0.035906
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique
</td>
<td style="text-align:right;">
0.003776
</td>
<td style="text-align:right;">
-0.721974
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.905575
</td>
<td style="text-align:right;">
-0.375058
</td>
<td style="text-align:right;">
-0.070040
</td>
<td style="text-align:right;">
-0.217942
</td>
<td style="text-align:right;">
-0.050508
</td>
<td style="text-align:right;">
0.316457
</td>
<td style="text-align:right;">
-0.144339
</td>
<td style="text-align:right;">
0.054071
</td>
<td style="text-align:right;">
-0.052375
</td>
<td style="text-align:right;">
-0.055582
</td>
<td style="text-align:right;">
0.057149
</td>
<td style="text-align:right;">
-0.067433
</td>
<td style="text-align:right;">
0.067262
</td>
<td style="text-align:right;">
-0.036631
</td>
<td style="text-align:right;">
-0.020556
</td>
<td style="text-align:right;">
-0.031947
</td>
<td style="text-align:right;">
0.036916
</td>
<td style="text-align:right;">
0.023104
</td>
<td style="text-align:right;">
0.041587
</td>
<td style="text-align:right;">
-0.116051
</td>
<td style="text-align:right;">
0.052917
</td>
<td style="text-align:right;">
0.008701
</td>
<td style="text-align:right;">
0.005021
</td>
<td style="text-align:right;">
0.109292
</td>
<td style="text-align:right;">
0.008649
</td>
<td style="text-align:right;">
-0.008961
</td>
<td style="text-align:right;">
-0.054646
</td>
<td style="text-align:right;">
-0.041530
</td>
<td style="text-align:right;">
0.098463
</td>
<td style="text-align:right;">
0.045140
</td>
<td style="text-align:right;">
0.052574
</td>
<td style="text-align:right;">
0.379052
</td>
<td style="text-align:right;">
-0.322923
</td>
<td style="text-align:right;">
0.049102
</td>
<td style="text-align:right;">
0.373979
</td>
<td style="text-align:right;">
-0.318527
</td>
<td style="text-align:right;">
-0.002959
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique.Nonstop
</td>
<td style="text-align:right;">
0.009661
</td>
<td style="text-align:right;">
-0.557612
</td>
<td style="text-align:right;">
0.905575
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.347247
</td>
<td style="text-align:right;">
-0.089376
</td>
<td style="text-align:right;">
-0.274584
</td>
<td style="text-align:right;">
-0.104847
</td>
<td style="text-align:right;">
0.341360
</td>
<td style="text-align:right;">
-0.136625
</td>
<td style="text-align:right;">
0.038076
</td>
<td style="text-align:right;">
-0.039402
</td>
<td style="text-align:right;">
-0.035763
</td>
<td style="text-align:right;">
0.041465
</td>
<td style="text-align:right;">
-0.051459
</td>
<td style="text-align:right;">
0.026054
</td>
<td style="text-align:right;">
-0.046502
</td>
<td style="text-align:right;">
-0.033905
</td>
<td style="text-align:right;">
-0.051320
</td>
<td style="text-align:right;">
0.027829
</td>
<td style="text-align:right;">
0.006429
</td>
<td style="text-align:right;">
0.026283
</td>
<td style="text-align:right;">
-0.051795
</td>
<td style="text-align:right;">
0.047018
</td>
<td style="text-align:right;">
0.018626
</td>
<td style="text-align:right;">
-0.066408
</td>
<td style="text-align:right;">
0.064462
</td>
<td style="text-align:right;">
0.114947
</td>
<td style="text-align:right;">
0.015631
</td>
<td style="text-align:right;">
0.009303
</td>
<td style="text-align:right;">
0.022318
</td>
<td style="text-align:right;">
0.128428
</td>
<td style="text-align:right;">
0.067471
</td>
<td style="text-align:right;">
0.122726
</td>
<td style="text-align:right;">
0.300479
</td>
<td style="text-align:right;">
-0.184595
</td>
<td style="text-align:right;">
-0.013454
</td>
<td style="text-align:right;">
0.227442
</td>
<td style="text-align:right;">
-0.246732
</td>
<td style="text-align:right;">
-0.008304
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Links
</td>
<td style="text-align:right;">
-0.058927
</td>
<td style="text-align:right;">
0.578959
</td>
<td style="text-align:right;">
-0.375058
</td>
<td style="text-align:right;">
-0.347247
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.352659
</td>
<td style="text-align:right;">
0.272617
</td>
<td style="text-align:right;">
0.163856
</td>
<td style="text-align:right;">
0.159750
</td>
<td style="text-align:right;">
0.203693
</td>
<td style="text-align:right;">
-0.080849
</td>
<td style="text-align:right;">
0.026927
</td>
<td style="text-align:right;">
0.024198
</td>
<td style="text-align:right;">
-0.047382
</td>
<td style="text-align:right;">
0.094664
</td>
<td style="text-align:right;">
-0.026355
</td>
<td style="text-align:right;">
0.042283
</td>
<td style="text-align:right;">
0.028325
</td>
<td style="text-align:right;">
0.076665
</td>
<td style="text-align:right;">
-0.004801
</td>
<td style="text-align:right;">
0.035151
</td>
<td style="text-align:right;">
0.003946
</td>
<td style="text-align:right;">
0.106647
</td>
<td style="text-align:right;">
-0.078813
</td>
<td style="text-align:right;">
-0.034563
</td>
<td style="text-align:right;">
-0.001623
</td>
<td style="text-align:right;">
-0.065198
</td>
<td style="text-align:right;">
0.113228
</td>
<td style="text-align:right;">
0.121085
</td>
<td style="text-align:right;">
0.132319
</td>
<td style="text-align:right;">
0.002315
</td>
<td style="text-align:right;">
0.087561
</td>
<td style="text-align:right;">
-0.060199
</td>
<td style="text-align:right;">
0.118563
</td>
<td style="text-align:right;">
-0.222707
</td>
<td style="text-align:right;">
0.329171
</td>
<td style="text-align:right;">
-0.106653
</td>
<td style="text-align:right;">
-0.281240
</td>
<td style="text-align:right;">
0.129478
</td>
<td style="text-align:right;">
0.031429
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Other
</td>
<td style="text-align:right;">
-0.023967
</td>
<td style="text-align:right;">
0.148974
</td>
<td style="text-align:right;">
-0.070040
</td>
<td style="text-align:right;">
-0.089376
</td>
<td style="text-align:right;">
0.352659
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.176709
</td>
<td style="text-align:right;">
0.048216
</td>
<td style="text-align:right;">
0.022879
</td>
<td style="text-align:right;">
0.030974
</td>
<td style="text-align:right;">
-0.084889
</td>
<td style="text-align:right;">
-0.018899
</td>
<td style="text-align:right;">
-0.052384
</td>
<td style="text-align:right;">
-0.017971
</td>
<td style="text-align:right;">
0.094539
</td>
<td style="text-align:right;">
0.054585
</td>
<td style="text-align:right;">
0.044219
</td>
<td style="text-align:right;">
-0.013071
</td>
<td style="text-align:right;">
0.008908
</td>
<td style="text-align:right;">
-0.026703
</td>
<td style="text-align:right;">
0.092482
</td>
<td style="text-align:right;">
0.022899
</td>
<td style="text-align:right;">
-0.058104
</td>
<td style="text-align:right;">
0.002679
</td>
<td style="text-align:right;">
-0.022965
</td>
<td style="text-align:right;">
-0.004764
</td>
<td style="text-align:right;">
0.092523
</td>
<td style="text-align:right;">
-0.009754
</td>
<td style="text-align:right;">
-0.025945
</td>
<td style="text-align:right;">
-0.025466
</td>
<td style="text-align:right;">
-0.007169
</td>
<td style="text-align:right;">
0.011748
</td>
<td style="text-align:right;">
0.014413
</td>
<td style="text-align:right;">
-0.014212
</td>
<td style="text-align:right;">
-0.041999
</td>
<td style="text-align:right;">
0.025987
</td>
<td style="text-align:right;">
-0.014408
</td>
<td style="text-align:right;">
-0.046358
</td>
<td style="text-align:right;">
0.008474
</td>
<td style="text-align:right;">
-0.029379
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Images
</td>
<td style="text-align:right;">
-0.023733
</td>
<td style="text-align:right;">
0.227858
</td>
<td style="text-align:right;">
-0.217942
</td>
<td style="text-align:right;">
-0.274584
</td>
<td style="text-align:right;">
0.272617
</td>
<td style="text-align:right;">
0.176709
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.014463
</td>
<td style="text-align:right;">
0.039445
</td>
<td style="text-align:right;">
0.090949
</td>
<td style="text-align:right;">
-0.011509
</td>
<td style="text-align:right;">
-0.009628
</td>
<td style="text-align:right;">
-0.020417
</td>
<td style="text-align:right;">
-0.015207
</td>
<td style="text-align:right;">
0.054040
</td>
<td style="text-align:right;">
0.003212
</td>
<td style="text-align:right;">
0.047418
</td>
<td style="text-align:right;">
0.001793
</td>
<td style="text-align:right;">
0.034160
</td>
<td style="text-align:right;">
-0.012836
</td>
<td style="text-align:right;">
-0.007416
</td>
<td style="text-align:right;">
-0.013635
</td>
<td style="text-align:right;">
-0.067741
</td>
<td style="text-align:right;">
0.009383
</td>
<td style="text-align:right;">
-0.006319
</td>
<td style="text-align:right;">
0.084308
</td>
<td style="text-align:right;">
0.037160
</td>
<td style="text-align:right;">
0.042410
</td>
<td style="text-align:right;">
0.007262
</td>
<td style="text-align:right;">
-0.032633
</td>
<td style="text-align:right;">
0.003014
</td>
<td style="text-align:right;">
-0.002960
</td>
<td style="text-align:right;">
0.017603
</td>
<td style="text-align:right;">
0.011254
</td>
<td style="text-align:right;">
-0.064787
</td>
<td style="text-align:right;">
0.098895
</td>
<td style="text-align:right;">
-0.017271
</td>
<td style="text-align:right;">
-0.090883
</td>
<td style="text-align:right;">
0.050848
</td>
<td style="text-align:right;">
0.010094
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Videos
</td>
<td style="text-align:right;">
0.042989
</td>
<td style="text-align:right;">
0.160154
</td>
<td style="text-align:right;">
-0.050508
</td>
<td style="text-align:right;">
-0.104847
</td>
<td style="text-align:right;">
0.163856
</td>
<td style="text-align:right;">
0.048216
</td>
<td style="text-align:right;">
-0.014463
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.029981
</td>
<td style="text-align:right;">
0.077666
</td>
<td style="text-align:right;">
0.038254
</td>
<td style="text-align:right;">
0.060059
</td>
<td style="text-align:right;">
0.035030
</td>
<td style="text-align:right;">
0.042253
</td>
<td style="text-align:right;">
-0.046000
</td>
<td style="text-align:right;">
0.068191
</td>
<td style="text-align:right;">
0.020928
</td>
<td style="text-align:right;">
0.064984
</td>
<td style="text-align:right;">
0.108978
</td>
<td style="text-align:right;">
-0.002195
</td>
<td style="text-align:right;">
0.098097
</td>
<td style="text-align:right;">
0.051158
</td>
<td style="text-align:right;">
0.005301
</td>
<td style="text-align:right;">
-0.030825
</td>
<td style="text-align:right;">
-0.050599
</td>
<td style="text-align:right;">
0.185670
</td>
<td style="text-align:right;">
-0.062353
</td>
<td style="text-align:right;">
0.068725
</td>
<td style="text-align:right;">
0.040273
</td>
<td style="text-align:right;">
0.101639
</td>
<td style="text-align:right;">
0.018369
</td>
<td style="text-align:right;">
0.028553
</td>
<td style="text-align:right;">
-0.030782
</td>
<td style="text-align:right;">
0.048939
</td>
<td style="text-align:right;">
-0.028524
</td>
<td style="text-align:right;">
0.085774
</td>
<td style="text-align:right;">
-0.101376
</td>
<td style="text-align:right;">
-0.097916
</td>
<td style="text-align:right;">
-0.001326
</td>
<td style="text-align:right;">
0.103662
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Words
</td>
<td style="text-align:right;">
-0.088000
</td>
<td style="text-align:right;">
0.026064
</td>
<td style="text-align:right;">
0.316457
</td>
<td style="text-align:right;">
0.341360
</td>
<td style="text-align:right;">
0.159750
</td>
<td style="text-align:right;">
0.022879
</td>
<td style="text-align:right;">
0.039445
</td>
<td style="text-align:right;">
-0.029981
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.024279
</td>
<td style="text-align:right;">
-0.022915
</td>
<td style="text-align:right;">
-0.007241
</td>
<td style="text-align:right;">
0.002464
</td>
<td style="text-align:right;">
-0.039639
</td>
<td style="text-align:right;">
0.015619
</td>
<td style="text-align:right;">
-0.053806
</td>
<td style="text-align:right;">
-0.034394
</td>
<td style="text-align:right;">
-0.093192
</td>
<td style="text-align:right;">
-0.109847
</td>
<td style="text-align:right;">
0.000211
</td>
<td style="text-align:right;">
-0.031833
</td>
<td style="text-align:right;">
-0.019280
</td>
<td style="text-align:right;">
0.075166
</td>
<td style="text-align:right;">
-0.024389
</td>
<td style="text-align:right;">
0.044189
</td>
<td style="text-align:right;">
-0.157070
</td>
<td style="text-align:right;">
-0.020442
</td>
<td style="text-align:right;">
0.141238
</td>
<td style="text-align:right;">
0.116943
</td>
<td style="text-align:right;">
0.133407
</td>
<td style="text-align:right;">
-0.041741
</td>
<td style="text-align:right;">
0.308729
</td>
<td style="text-align:right;">
0.003704
</td>
<td style="text-align:right;">
0.154391
</td>
<td style="text-align:right;">
0.008935
</td>
<td style="text-align:right;">
0.157859
</td>
<td style="text-align:right;">
-0.040404
</td>
<td style="text-align:right;">
0.015285
</td>
<td style="text-align:right;">
-0.074783
</td>
<td style="text-align:right;">
-0.033952
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Key
</td>
<td style="text-align:right;">
-0.002771
</td>
<td style="text-align:right;">
0.167237
</td>
<td style="text-align:right;">
-0.144339
</td>
<td style="text-align:right;">
-0.136625
</td>
<td style="text-align:right;">
0.203693
</td>
<td style="text-align:right;">
0.030974
</td>
<td style="text-align:right;">
0.090949
</td>
<td style="text-align:right;">
0.077666
</td>
<td style="text-align:right;">
-0.024279
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.026127
</td>
<td style="text-align:right;">
0.118038
</td>
<td style="text-align:right;">
0.135322
</td>
<td style="text-align:right;">
-0.272399
</td>
<td style="text-align:right;">
-0.019084
</td>
<td style="text-align:right;">
-0.414678
</td>
<td style="text-align:right;">
-0.349029
</td>
<td style="text-align:right;">
0.102702
</td>
<td style="text-align:right;">
-0.000065
</td>
<td style="text-align:right;">
0.000245
</td>
<td style="text-align:right;">
0.020112
</td>
<td style="text-align:right;">
0.014568
</td>
<td style="text-align:right;">
-0.097906
</td>
<td style="text-align:right;">
0.010482
</td>
<td style="text-align:right;">
0.024135
</td>
<td style="text-align:right;">
0.044276
</td>
<td style="text-align:right;">
0.078602
</td>
<td style="text-align:right;">
0.064004
</td>
<td style="text-align:right;">
0.111318
</td>
<td style="text-align:right;">
0.125501
</td>
<td style="text-align:right;">
-0.028257
</td>
<td style="text-align:right;">
0.071565
</td>
<td style="text-align:right;">
-0.103740
</td>
<td style="text-align:right;">
0.043282
</td>
<td style="text-align:right;">
-0.105683
</td>
<td style="text-align:right;">
0.141371
</td>
<td style="text-align:right;">
-0.016445
</td>
<td style="text-align:right;">
-0.068220
</td>
<td style="text-align:right;">
0.048973
</td>
<td style="text-align:right;">
0.031114
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Worst.Key
</td>
<td style="text-align:right;">
-0.045687
</td>
<td style="text-align:right;">
-0.065708
</td>
<td style="text-align:right;">
0.054071
</td>
<td style="text-align:right;">
0.038076
</td>
<td style="text-align:right;">
-0.080849
</td>
<td style="text-align:right;">
-0.084889
</td>
<td style="text-align:right;">
-0.011509
</td>
<td style="text-align:right;">
0.038254
</td>
<td style="text-align:right;">
-0.022915
</td>
<td style="text-align:right;">
0.026127
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.003657
</td>
<td style="text-align:right;">
0.161406
</td>
<td style="text-align:right;">
-0.076830
</td>
<td style="text-align:right;">
-0.859758
</td>
<td style="text-align:right;">
-0.646789
</td>
<td style="text-align:right;">
-0.187259
</td>
<td style="text-align:right;">
-0.071779
</td>
<td style="text-align:right;">
-0.208502
</td>
<td style="text-align:right;">
-0.030486
</td>
<td style="text-align:right;">
-0.054840
</td>
<td style="text-align:right;">
-0.049069
</td>
<td style="text-align:right;">
-0.018374
</td>
<td style="text-align:right;">
0.055944
</td>
<td style="text-align:right;">
-0.030303
</td>
<td style="text-align:right;">
-0.041323
</td>
<td style="text-align:right;">
0.033874
</td>
<td style="text-align:right;">
-0.021358
</td>
<td style="text-align:right;">
0.041154
</td>
<td style="text-align:right;">
0.048226
</td>
<td style="text-align:right;">
-0.030342
</td>
<td style="text-align:right;">
0.037036
</td>
<td style="text-align:right;">
-0.046518
</td>
<td style="text-align:right;">
-0.016350
</td>
<td style="text-align:right;">
0.005734
</td>
<td style="text-align:right;">
-0.039842
</td>
<td style="text-align:right;">
0.064335
</td>
<td style="text-align:right;">
0.077298
</td>
<td style="text-align:right;">
-0.013131
</td>
<td style="text-align:right;">
0.008704
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Worst.Key
</td>
<td style="text-align:right;">
-0.024068
</td>
<td style="text-align:right;">
0.061507
</td>
<td style="text-align:right;">
-0.052375
</td>
<td style="text-align:right;">
-0.039402
</td>
<td style="text-align:right;">
0.026927
</td>
<td style="text-align:right;">
-0.018899
</td>
<td style="text-align:right;">
-0.009628
</td>
<td style="text-align:right;">
0.060059
</td>
<td style="text-align:right;">
-0.007241
</td>
<td style="text-align:right;">
0.118038
</td>
<td style="text-align:right;">
0.003657
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.935909
</td>
<td style="text-align:right;">
-0.059514
</td>
<td style="text-align:right;">
-0.001202
</td>
<td style="text-align:right;">
-0.066126
</td>
<td style="text-align:right;">
0.004583
</td>
<td style="text-align:right;">
0.281468
</td>
<td style="text-align:right;">
0.251209
</td>
<td style="text-align:right;">
0.027103
</td>
<td style="text-align:right;">
0.146831
</td>
<td style="text-align:right;">
0.101141
</td>
<td style="text-align:right;">
0.010485
</td>
<td style="text-align:right;">
0.006853
</td>
<td style="text-align:right;">
-0.000680
</td>
<td style="text-align:right;">
0.031244
</td>
<td style="text-align:right;">
-0.035876
</td>
<td style="text-align:right;">
0.061498
</td>
<td style="text-align:right;">
0.013305
</td>
<td style="text-align:right;">
0.026836
</td>
<td style="text-align:right;">
0.004767
</td>
<td style="text-align:right;">
0.008048
</td>
<td style="text-align:right;">
-0.007472
</td>
<td style="text-align:right;">
0.023270
</td>
<td style="text-align:right;">
-0.021196
</td>
<td style="text-align:right;">
0.039729
</td>
<td style="text-align:right;">
-0.050842
</td>
<td style="text-align:right;">
-0.057745
</td>
<td style="text-align:right;">
-0.000680
</td>
<td style="text-align:right;">
0.003850
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Worst.Key
</td>
<td style="text-align:right;">
-0.057209
</td>
<td style="text-align:right;">
0.062479
</td>
<td style="text-align:right;">
-0.055582
</td>
<td style="text-align:right;">
-0.035763
</td>
<td style="text-align:right;">
0.024198
</td>
<td style="text-align:right;">
-0.052384
</td>
<td style="text-align:right;">
-0.020417
</td>
<td style="text-align:right;">
0.035030
</td>
<td style="text-align:right;">
0.002464
</td>
<td style="text-align:right;">
0.135322
</td>
<td style="text-align:right;">
0.161406
</td>
<td style="text-align:right;">
0.935909
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.107277
</td>
<td style="text-align:right;">
-0.156618
</td>
<td style="text-align:right;">
-0.228252
</td>
<td style="text-align:right;">
-0.048625
</td>
<td style="text-align:right;">
0.252130
</td>
<td style="text-align:right;">
0.205284
</td>
<td style="text-align:right;">
0.039203
</td>
<td style="text-align:right;">
0.105069
</td>
<td style="text-align:right;">
0.083423
</td>
<td style="text-align:right;">
0.031241
</td>
<td style="text-align:right;">
0.027170
</td>
<td style="text-align:right;">
-0.012238
</td>
<td style="text-align:right;">
-0.011088
</td>
<td style="text-align:right;">
-0.042886
</td>
<td style="text-align:right;">
0.054498
</td>
<td style="text-align:right;">
0.024189
</td>
<td style="text-align:right;">
0.046949
</td>
<td style="text-align:right;">
0.005268
</td>
<td style="text-align:right;">
0.021085
</td>
<td style="text-align:right;">
-0.016061
</td>
<td style="text-align:right;">
0.024010
</td>
<td style="text-align:right;">
-0.036495
</td>
<td style="text-align:right;">
0.045633
</td>
<td style="text-align:right;">
-0.040548
</td>
<td style="text-align:right;">
-0.052646
</td>
<td style="text-align:right;">
0.000405
</td>
<td style="text-align:right;">
-0.007586
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Best.Key
</td>
<td style="text-align:right;">
0.035840
</td>
<td style="text-align:right;">
-0.050350
</td>
<td style="text-align:right;">
0.057149
</td>
<td style="text-align:right;">
0.041465
</td>
<td style="text-align:right;">
-0.047382
</td>
<td style="text-align:right;">
-0.017971
</td>
<td style="text-align:right;">
-0.015207
</td>
<td style="text-align:right;">
0.042253
</td>
<td style="text-align:right;">
-0.039639
</td>
<td style="text-align:right;">
-0.272399
</td>
<td style="text-align:right;">
-0.076830
</td>
<td style="text-align:right;">
-0.059514
</td>
<td style="text-align:right;">
-0.107277
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.080416
</td>
<td style="text-align:right;">
0.454592
</td>
<td style="text-align:right;">
0.385403
</td>
<td style="text-align:right;">
0.056699
</td>
<td style="text-align:right;">
0.197143
</td>
<td style="text-align:right;">
-0.000593
</td>
<td style="text-align:right;">
0.057843
</td>
<td style="text-align:right;">
0.027145
</td>
<td style="text-align:right;">
0.037898
</td>
<td style="text-align:right;">
-0.027671
</td>
<td style="text-align:right;">
-0.023632
</td>
<td style="text-align:right;">
0.042332
</td>
<td style="text-align:right;">
-0.040886
</td>
<td style="text-align:right;">
0.009652
</td>
<td style="text-align:right;">
-0.032486
</td>
<td style="text-align:right;">
-0.013096
</td>
<td style="text-align:right;">
0.014274
</td>
<td style="text-align:right;">
-0.024577
</td>
<td style="text-align:right;">
0.016915
</td>
<td style="text-align:right;">
0.010245
</td>
<td style="text-align:right;">
0.061832
</td>
<td style="text-align:right;">
-0.031010
</td>
<td style="text-align:right;">
-0.034181
</td>
<td style="text-align:right;">
-0.010573
</td>
<td style="text-align:right;">
-0.034984
</td>
<td style="text-align:right;">
0.032256
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Best.Key
</td>
<td style="text-align:right;">
0.055915
</td>
<td style="text-align:right;">
0.077450
</td>
<td style="text-align:right;">
-0.067433
</td>
<td style="text-align:right;">
-0.051459
</td>
<td style="text-align:right;">
0.094664
</td>
<td style="text-align:right;">
0.094539
</td>
<td style="text-align:right;">
0.054040
</td>
<td style="text-align:right;">
-0.046000
</td>
<td style="text-align:right;">
0.015619
</td>
<td style="text-align:right;">
-0.019084
</td>
<td style="text-align:right;">
-0.859758
</td>
<td style="text-align:right;">
-0.001202
</td>
<td style="text-align:right;">
-0.156618
</td>
<td style="text-align:right;">
0.080416
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.670122
</td>
<td style="text-align:right;">
0.206761
</td>
<td style="text-align:right;">
0.080236
</td>
<td style="text-align:right;">
0.232083
</td>
<td style="text-align:right;">
0.029514
</td>
<td style="text-align:right;">
0.048358
</td>
<td style="text-align:right;">
0.044865
</td>
<td style="text-align:right;">
0.018316
</td>
<td style="text-align:right;">
-0.052492
</td>
<td style="text-align:right;">
0.026723
</td>
<td style="text-align:right;">
0.050406
</td>
<td style="text-align:right;">
-0.038865
</td>
<td style="text-align:right;">
0.021083
</td>
<td style="text-align:right;">
-0.043219
</td>
<td style="text-align:right;">
-0.053481
</td>
<td style="text-align:right;">
0.016672
</td>
<td style="text-align:right;">
-0.032509
</td>
<td style="text-align:right;">
0.035711
</td>
<td style="text-align:right;">
0.006070
</td>
<td style="text-align:right;">
-0.013973
</td>
<td style="text-align:right;">
0.035732
</td>
<td style="text-align:right;">
-0.065829
</td>
<td style="text-align:right;">
-0.083724
</td>
<td style="text-align:right;">
0.015733
</td>
<td style="text-align:right;">
-0.014187
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Best.Key
</td>
<td style="text-align:right;">
0.056975
</td>
<td style="text-align:right;">
-0.077576
</td>
<td style="text-align:right;">
0.067262
</td>
<td style="text-align:right;">
0.026054
</td>
<td style="text-align:right;">
-0.026355
</td>
<td style="text-align:right;">
0.054585
</td>
<td style="text-align:right;">
0.003212
</td>
<td style="text-align:right;">
0.068191
</td>
<td style="text-align:right;">
-0.053806
</td>
<td style="text-align:right;">
-0.414678
</td>
<td style="text-align:right;">
-0.646789
</td>
<td style="text-align:right;">
-0.066126
</td>
<td style="text-align:right;">
-0.228252
</td>
<td style="text-align:right;">
0.454592
</td>
<td style="text-align:right;">
0.670122
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.470605
</td>
<td style="text-align:right;">
0.122160
</td>
<td style="text-align:right;">
0.373956
</td>
<td style="text-align:right;">
0.052408
</td>
<td style="text-align:right;">
0.121327
</td>
<td style="text-align:right;">
0.098607
</td>
<td style="text-align:right;">
0.040423
</td>
<td style="text-align:right;">
-0.043973
</td>
<td style="text-align:right;">
-0.015503
</td>
<td style="text-align:right;">
0.181910
</td>
<td style="text-align:right;">
-0.121607
</td>
<td style="text-align:right;">
0.004240
</td>
<td style="text-align:right;">
-0.067464
</td>
<td style="text-align:right;">
-0.072628
</td>
<td style="text-align:right;">
0.013668
</td>
<td style="text-align:right;">
-0.060556
</td>
<td style="text-align:right;">
0.041416
</td>
<td style="text-align:right;">
-0.010076
</td>
<td style="text-align:right;">
0.049089
</td>
<td style="text-align:right;">
-0.058639
</td>
<td style="text-align:right;">
-0.081052
</td>
<td style="text-align:right;">
-0.028571
</td>
<td style="text-align:right;">
-0.063902
</td>
<td style="text-align:right;">
0.039557
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Min.Key
</td>
<td style="text-align:right;">
0.019985
</td>
<td style="text-align:right;">
0.023457
</td>
<td style="text-align:right;">
-0.036631
</td>
<td style="text-align:right;">
-0.046502
</td>
<td style="text-align:right;">
0.042283
</td>
<td style="text-align:right;">
0.044219
</td>
<td style="text-align:right;">
0.047418
</td>
<td style="text-align:right;">
0.020928
</td>
<td style="text-align:right;">
-0.034394
</td>
<td style="text-align:right;">
-0.349029
</td>
<td style="text-align:right;">
-0.187259
</td>
<td style="text-align:right;">
0.004583
</td>
<td style="text-align:right;">
-0.048625
</td>
<td style="text-align:right;">
0.385403
</td>
<td style="text-align:right;">
0.206761
</td>
<td style="text-align:right;">
0.470605
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.054344
</td>
<td style="text-align:right;">
0.360764
</td>
<td style="text-align:right;">
0.018640
</td>
<td style="text-align:right;">
0.060192
</td>
<td style="text-align:right;">
0.047023
</td>
<td style="text-align:right;">
0.001623
</td>
<td style="text-align:right;">
-0.053641
</td>
<td style="text-align:right;">
-0.043644
</td>
<td style="text-align:right;">
0.100943
</td>
<td style="text-align:right;">
0.001827
</td>
<td style="text-align:right;">
0.042448
</td>
<td style="text-align:right;">
0.011730
</td>
<td style="text-align:right;">
0.029399
</td>
<td style="text-align:right;">
0.017326
</td>
<td style="text-align:right;">
-0.007391
</td>
<td style="text-align:right;">
-0.000731
</td>
<td style="text-align:right;">
0.022243
</td>
<td style="text-align:right;">
-0.006112
</td>
<td style="text-align:right;">
0.015075
</td>
<td style="text-align:right;">
-0.034594
</td>
<td style="text-align:right;">
-0.038540
</td>
<td style="text-align:right;">
-0.000435
</td>
<td style="text-align:right;">
0.031850
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Max.Key
</td>
<td style="text-align:right;">
0.028279
</td>
<td style="text-align:right;">
0.012473
</td>
<td style="text-align:right;">
-0.020556
</td>
<td style="text-align:right;">
-0.033905
</td>
<td style="text-align:right;">
0.028325
</td>
<td style="text-align:right;">
-0.013071
</td>
<td style="text-align:right;">
0.001793
</td>
<td style="text-align:right;">
0.064984
</td>
<td style="text-align:right;">
-0.093192
</td>
<td style="text-align:right;">
0.102702
</td>
<td style="text-align:right;">
-0.071779
</td>
<td style="text-align:right;">
0.281468
</td>
<td style="text-align:right;">
0.252130
</td>
<td style="text-align:right;">
0.056699
</td>
<td style="text-align:right;">
0.080236
</td>
<td style="text-align:right;">
0.122160
</td>
<td style="text-align:right;">
0.054344
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.876213
</td>
<td style="text-align:right;">
0.183550
</td>
<td style="text-align:right;">
0.368978
</td>
<td style="text-align:right;">
0.306684
</td>
<td style="text-align:right;">
0.013534
</td>
<td style="text-align:right;">
0.011134
</td>
<td style="text-align:right;">
-0.032329
</td>
<td style="text-align:right;">
0.109981
</td>
<td style="text-align:right;">
-0.067285
</td>
<td style="text-align:right;">
0.050830
</td>
<td style="text-align:right;">
0.012144
</td>
<td style="text-align:right;">
0.033077
</td>
<td style="text-align:right;">
0.019672
</td>
<td style="text-align:right;">
-0.018928
</td>
<td style="text-align:right;">
-0.010139
</td>
<td style="text-align:right;">
0.018977
</td>
<td style="text-align:right;">
-0.021240
</td>
<td style="text-align:right;">
0.015943
</td>
<td style="text-align:right;">
-0.072157
</td>
<td style="text-align:right;">
-0.063959
</td>
<td style="text-align:right;">
-0.021721
</td>
<td style="text-align:right;">
0.051564
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Avg.Key
</td>
<td style="text-align:right;">
0.035345
</td>
<td style="text-align:right;">
0.039085
</td>
<td style="text-align:right;">
-0.031947
</td>
<td style="text-align:right;">
-0.051320
</td>
<td style="text-align:right;">
0.076665
</td>
<td style="text-align:right;">
0.008908
</td>
<td style="text-align:right;">
0.034160
</td>
<td style="text-align:right;">
0.108978
</td>
<td style="text-align:right;">
-0.109847
</td>
<td style="text-align:right;">
-0.000065
</td>
<td style="text-align:right;">
-0.208502
</td>
<td style="text-align:right;">
0.251209
</td>
<td style="text-align:right;">
0.205284
</td>
<td style="text-align:right;">
0.197143
</td>
<td style="text-align:right;">
0.232083
</td>
<td style="text-align:right;">
0.373956
</td>
<td style="text-align:right;">
0.360764
</td>
<td style="text-align:right;">
0.876213
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.161337
</td>
<td style="text-align:right;">
0.358635
</td>
<td style="text-align:right;">
0.287962
</td>
<td style="text-align:right;">
0.027752
</td>
<td style="text-align:right;">
-0.017178
</td>
<td style="text-align:right;">
-0.073452
</td>
<td style="text-align:right;">
0.192805
</td>
<td style="text-align:right;">
-0.088799
</td>
<td style="text-align:right;">
0.089336
</td>
<td style="text-align:right;">
0.017802
</td>
<td style="text-align:right;">
0.053369
</td>
<td style="text-align:right;">
0.038720
</td>
<td style="text-align:right;">
-0.026407
</td>
<td style="text-align:right;">
-0.007812
</td>
<td style="text-align:right;">
0.040306
</td>
<td style="text-align:right;">
-0.026120
</td>
<td style="text-align:right;">
0.035835
</td>
<td style="text-align:right;">
-0.102511
</td>
<td style="text-align:right;">
-0.093344
</td>
<td style="text-align:right;">
-0.029889
</td>
<td style="text-align:right;">
0.072562
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Ref
</td>
<td style="text-align:right;">
-0.006686
</td>
<td style="text-align:right;">
-0.024377
</td>
<td style="text-align:right;">
0.036916
</td>
<td style="text-align:right;">
0.027829
</td>
<td style="text-align:right;">
-0.004801
</td>
<td style="text-align:right;">
-0.026703
</td>
<td style="text-align:right;">
-0.012836
</td>
<td style="text-align:right;">
-0.002195
</td>
<td style="text-align:right;">
0.000211
</td>
<td style="text-align:right;">
0.000245
</td>
<td style="text-align:right;">
-0.030486
</td>
<td style="text-align:right;">
0.027103
</td>
<td style="text-align:right;">
0.039203
</td>
<td style="text-align:right;">
-0.000593
</td>
<td style="text-align:right;">
0.029514
</td>
<td style="text-align:right;">
0.052408
</td>
<td style="text-align:right;">
0.018640
</td>
<td style="text-align:right;">
0.183550
</td>
<td style="text-align:right;">
0.161337
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.497681
</td>
<td style="text-align:right;">
0.826240
</td>
<td style="text-align:right;">
0.006528
</td>
<td style="text-align:right;">
0.011988
</td>
<td style="text-align:right;">
-0.010124
</td>
<td style="text-align:right;">
0.034650
</td>
<td style="text-align:right;">
-0.029502
</td>
<td style="text-align:right;">
0.039435
</td>
<td style="text-align:right;">
-0.018593
</td>
<td style="text-align:right;">
-0.033334
</td>
<td style="text-align:right;">
-0.016519
</td>
<td style="text-align:right;">
0.005658
</td>
<td style="text-align:right;">
-0.001880
</td>
<td style="text-align:right;">
0.000572
</td>
<td style="text-align:right;">
0.008484
</td>
<td style="text-align:right;">
-0.027895
</td>
<td style="text-align:right;">
-0.078516
</td>
<td style="text-align:right;">
-0.019040
</td>
<td style="text-align:right;">
-0.109572
</td>
<td style="text-align:right;">
0.001408
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Ref
</td>
<td style="text-align:right;">
0.001985
</td>
<td style="text-align:right;">
0.002683
</td>
<td style="text-align:right;">
0.023104
</td>
<td style="text-align:right;">
0.006429
</td>
<td style="text-align:right;">
0.035151
</td>
<td style="text-align:right;">
0.092482
</td>
<td style="text-align:right;">
-0.007416
</td>
<td style="text-align:right;">
0.098097
</td>
<td style="text-align:right;">
-0.031833
</td>
<td style="text-align:right;">
0.020112
</td>
<td style="text-align:right;">
-0.054840
</td>
<td style="text-align:right;">
0.146831
</td>
<td style="text-align:right;">
0.105069
</td>
<td style="text-align:right;">
0.057843
</td>
<td style="text-align:right;">
0.048358
</td>
<td style="text-align:right;">
0.121327
</td>
<td style="text-align:right;">
0.060192
</td>
<td style="text-align:right;">
0.368978
</td>
<td style="text-align:right;">
0.358635
</td>
<td style="text-align:right;">
0.497681
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.869974
</td>
<td style="text-align:right;">
-0.021773
</td>
<td style="text-align:right;">
0.013044
</td>
<td style="text-align:right;">
-0.000827
</td>
<td style="text-align:right;">
0.102307
</td>
<td style="text-align:right;">
-0.039394
</td>
<td style="text-align:right;">
0.055510
</td>
<td style="text-align:right;">
-0.003695
</td>
<td style="text-align:right;">
0.013926
</td>
<td style="text-align:right;">
0.009229
</td>
<td style="text-align:right;">
0.008616
</td>
<td style="text-align:right;">
-0.004850
</td>
<td style="text-align:right;">
0.028924
</td>
<td style="text-align:right;">
-0.009294
</td>
<td style="text-align:right;">
0.010761
</td>
<td style="text-align:right;">
-0.087390
</td>
<td style="text-align:right;">
-0.053293
</td>
<td style="text-align:right;">
-0.065351
</td>
<td style="text-align:right;">
0.050487
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Ref
</td>
<td style="text-align:right;">
0.002322
</td>
<td style="text-align:right;">
-0.021404
</td>
<td style="text-align:right;">
0.041587
</td>
<td style="text-align:right;">
0.026283
</td>
<td style="text-align:right;">
0.003946
</td>
<td style="text-align:right;">
0.022899
</td>
<td style="text-align:right;">
-0.013635
</td>
<td style="text-align:right;">
0.051158
</td>
<td style="text-align:right;">
-0.019280
</td>
<td style="text-align:right;">
0.014568
</td>
<td style="text-align:right;">
-0.049069
</td>
<td style="text-align:right;">
0.101141
</td>
<td style="text-align:right;">
0.083423
</td>
<td style="text-align:right;">
0.027145
</td>
<td style="text-align:right;">
0.044865
</td>
<td style="text-align:right;">
0.098607
</td>
<td style="text-align:right;">
0.047023
</td>
<td style="text-align:right;">
0.306684
</td>
<td style="text-align:right;">
0.287962
</td>
<td style="text-align:right;">
0.826240
</td>
<td style="text-align:right;">
0.869974
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.013942
</td>
<td style="text-align:right;">
0.012844
</td>
<td style="text-align:right;">
0.002373
</td>
<td style="text-align:right;">
0.080532
</td>
<td style="text-align:right;">
-0.038850
</td>
<td style="text-align:right;">
0.052788
</td>
<td style="text-align:right;">
-0.009756
</td>
<td style="text-align:right;">
-0.007544
</td>
<td style="text-align:right;">
-0.008681
</td>
<td style="text-align:right;">
0.012976
</td>
<td style="text-align:right;">
-0.009279
</td>
<td style="text-align:right;">
0.013567
</td>
<td style="text-align:right;">
-0.000161
</td>
<td style="text-align:right;">
-0.009345
</td>
<td style="text-align:right;">
-0.089864
</td>
<td style="text-align:right;">
-0.033144
</td>
<td style="text-align:right;">
-0.101930
</td>
<td style="text-align:right;">
0.030205
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_00
</td>
<td style="text-align:right;">
-0.071265
</td>
<td style="text-align:right;">
0.152093
</td>
<td style="text-align:right;">
-0.116051
</td>
<td style="text-align:right;">
-0.051795
</td>
<td style="text-align:right;">
0.106647
</td>
<td style="text-align:right;">
-0.058104
</td>
<td style="text-align:right;">
-0.067741
</td>
<td style="text-align:right;">
0.005301
</td>
<td style="text-align:right;">
0.075166
</td>
<td style="text-align:right;">
-0.097906
</td>
<td style="text-align:right;">
-0.018374
</td>
<td style="text-align:right;">
0.010485
</td>
<td style="text-align:right;">
0.031241
</td>
<td style="text-align:right;">
0.037898
</td>
<td style="text-align:right;">
0.018316
</td>
<td style="text-align:right;">
0.040423
</td>
<td style="text-align:right;">
0.001623
</td>
<td style="text-align:right;">
0.013534
</td>
<td style="text-align:right;">
0.027752
</td>
<td style="text-align:right;">
0.006528
</td>
<td style="text-align:right;">
-0.021773
</td>
<td style="text-align:right;">
-0.013942
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.325201
</td>
<td style="text-align:right;">
-0.419231
</td>
<td style="text-align:right;">
-0.296803
</td>
<td style="text-align:right;">
-0.640016
</td>
<td style="text-align:right;">
0.021474
</td>
<td style="text-align:right;">
0.121511
</td>
<td style="text-align:right;">
0.184908
</td>
<td style="text-align:right;">
-0.010781
</td>
<td style="text-align:right;">
0.122006
</td>
<td style="text-align:right;">
-0.109006
</td>
<td style="text-align:right;">
0.025086
</td>
<td style="text-align:right;">
-0.150526
</td>
<td style="text-align:right;">
0.134316
</td>
<td style="text-align:right;">
-0.017826
</td>
<td style="text-align:right;">
-0.064247
</td>
<td style="text-align:right;">
0.033366
</td>
<td style="text-align:right;">
-0.002494
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_01
</td>
<td style="text-align:right;">
0.002299
</td>
<td style="text-align:right;">
-0.069704
</td>
<td style="text-align:right;">
0.052917
</td>
<td style="text-align:right;">
0.047018
</td>
<td style="text-align:right;">
-0.078813
</td>
<td style="text-align:right;">
0.002679
</td>
<td style="text-align:right;">
0.009383
</td>
<td style="text-align:right;">
-0.030825
</td>
<td style="text-align:right;">
-0.024389
</td>
<td style="text-align:right;">
0.010482
</td>
<td style="text-align:right;">
0.055944
</td>
<td style="text-align:right;">
0.006853
</td>
<td style="text-align:right;">
0.027170
</td>
<td style="text-align:right;">
-0.027671
</td>
<td style="text-align:right;">
-0.052492
</td>
<td style="text-align:right;">
-0.043973
</td>
<td style="text-align:right;">
-0.053641
</td>
<td style="text-align:right;">
0.011134
</td>
<td style="text-align:right;">
-0.017178
</td>
<td style="text-align:right;">
0.011988
</td>
<td style="text-align:right;">
0.013044
</td>
<td style="text-align:right;">
0.012844
</td>
<td style="text-align:right;">
-0.325201
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.092426
</td>
<td style="text-align:right;">
-0.051683
</td>
<td style="text-align:right;">
-0.108793
</td>
<td style="text-align:right;">
-0.012861
</td>
<td style="text-align:right;">
-0.032922
</td>
<td style="text-align:right;">
-0.050156
</td>
<td style="text-align:right;">
0.010791
</td>
<td style="text-align:right;">
-0.019218
</td>
<td style="text-align:right;">
0.020336
</td>
<td style="text-align:right;">
-0.007544
</td>
<td style="text-align:right;">
0.035223
</td>
<td style="text-align:right;">
-0.042163
</td>
<td style="text-align:right;">
-0.006777
</td>
<td style="text-align:right;">
0.034024
</td>
<td style="text-align:right;">
-0.029982
</td>
<td style="text-align:right;">
-0.017964
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_02
</td>
<td style="text-align:right;">
0.050126
</td>
<td style="text-align:right;">
-0.004284
</td>
<td style="text-align:right;">
0.008701
</td>
<td style="text-align:right;">
0.018626
</td>
<td style="text-align:right;">
-0.034563
</td>
<td style="text-align:right;">
-0.022965
</td>
<td style="text-align:right;">
-0.006319
</td>
<td style="text-align:right;">
-0.050599
</td>
<td style="text-align:right;">
0.044189
</td>
<td style="text-align:right;">
0.024135
</td>
<td style="text-align:right;">
-0.030303
</td>
<td style="text-align:right;">
-0.000680
</td>
<td style="text-align:right;">
-0.012238
</td>
<td style="text-align:right;">
-0.023632
</td>
<td style="text-align:right;">
0.026723
</td>
<td style="text-align:right;">
-0.015503
</td>
<td style="text-align:right;">
-0.043644
</td>
<td style="text-align:right;">
-0.032329
</td>
<td style="text-align:right;">
-0.073452
</td>
<td style="text-align:right;">
-0.010124
</td>
<td style="text-align:right;">
-0.000827
</td>
<td style="text-align:right;">
0.002373
</td>
<td style="text-align:right;">
-0.419231
</td>
<td style="text-align:right;">
-0.092426
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.067574
</td>
<td style="text-align:right;">
-0.036973
</td>
<td style="text-align:right;">
-0.045286
</td>
<td style="text-align:right;">
-0.118931
</td>
<td style="text-align:right;">
-0.126306
</td>
<td style="text-align:right;">
0.031634
</td>
<td style="text-align:right;">
-0.081815
</td>
<td style="text-align:right;">
0.100592
</td>
<td style="text-align:right;">
-0.049298
</td>
<td style="text-align:right;">
0.031953
</td>
<td style="text-align:right;">
-0.075766
</td>
<td style="text-align:right;">
-0.006558
</td>
<td style="text-align:right;">
-0.021705
</td>
<td style="text-align:right;">
0.010383
</td>
<td style="text-align:right;">
-0.035788
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_03
</td>
<td style="text-align:right;">
0.028309
</td>
<td style="text-align:right;">
-0.065903
</td>
<td style="text-align:right;">
0.005021
</td>
<td style="text-align:right;">
-0.066408
</td>
<td style="text-align:right;">
-0.001623
</td>
<td style="text-align:right;">
-0.004764
</td>
<td style="text-align:right;">
0.084308
</td>
<td style="text-align:right;">
0.185670
</td>
<td style="text-align:right;">
-0.157070
</td>
<td style="text-align:right;">
0.044276
</td>
<td style="text-align:right;">
-0.041323
</td>
<td style="text-align:right;">
0.031244
</td>
<td style="text-align:right;">
-0.011088
</td>
<td style="text-align:right;">
0.042332
</td>
<td style="text-align:right;">
0.050406
</td>
<td style="text-align:right;">
0.181910
</td>
<td style="text-align:right;">
0.100943
</td>
<td style="text-align:right;">
0.109981
</td>
<td style="text-align:right;">
0.192805
</td>
<td style="text-align:right;">
0.034650
</td>
<td style="text-align:right;">
0.102307
</td>
<td style="text-align:right;">
0.080532
</td>
<td style="text-align:right;">
-0.296803
</td>
<td style="text-align:right;">
-0.051683
</td>
<td style="text-align:right;">
-0.067574
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.122832
</td>
<td style="text-align:right;">
0.051743
</td>
<td style="text-align:right;">
-0.051799
</td>
<td style="text-align:right;">
-0.067472
</td>
<td style="text-align:right;">
0.012031
</td>
<td style="text-align:right;">
-0.097945
</td>
<td style="text-align:right;">
0.024276
</td>
<td style="text-align:right;">
-0.002969
</td>
<td style="text-align:right;">
0.033049
</td>
<td style="text-align:right;">
-0.044340
</td>
<td style="text-align:right;">
-0.065233
</td>
<td style="text-align:right;">
-0.020370
</td>
<td style="text-align:right;">
-0.038350
</td>
<td style="text-align:right;">
0.087760
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_04
</td>
<td style="text-align:right;">
0.040678
</td>
<td style="text-align:right;">
-0.112971
</td>
<td style="text-align:right;">
0.109292
</td>
<td style="text-align:right;">
0.064462
</td>
<td style="text-align:right;">
-0.065198
</td>
<td style="text-align:right;">
0.092523
</td>
<td style="text-align:right;">
0.037160
</td>
<td style="text-align:right;">
-0.062353
</td>
<td style="text-align:right;">
-0.020442
</td>
<td style="text-align:right;">
0.078602
</td>
<td style="text-align:right;">
0.033874
</td>
<td style="text-align:right;">
-0.035876
</td>
<td style="text-align:right;">
-0.042886
</td>
<td style="text-align:right;">
-0.040886
</td>
<td style="text-align:right;">
-0.038865
</td>
<td style="text-align:right;">
-0.121607
</td>
<td style="text-align:right;">
0.001827
</td>
<td style="text-align:right;">
-0.067285
</td>
<td style="text-align:right;">
-0.088799
</td>
<td style="text-align:right;">
-0.029502
</td>
<td style="text-align:right;">
-0.039394
</td>
<td style="text-align:right;">
-0.038850
</td>
<td style="text-align:right;">
-0.640016
</td>
<td style="text-align:right;">
-0.108793
</td>
<td style="text-align:right;">
-0.036973
</td>
<td style="text-align:right;">
-0.122832
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.019532
</td>
<td style="text-align:right;">
-0.026329
</td>
<td style="text-align:right;">
-0.083953
</td>
<td style="text-align:right;">
-0.021326
</td>
<td style="text-align:right;">
-0.033696
</td>
<td style="text-align:right;">
0.046588
</td>
<td style="text-align:right;">
0.007343
</td>
<td style="text-align:right;">
0.132995
</td>
<td style="text-align:right;">
-0.071029
</td>
<td style="text-align:right;">
0.070322
</td>
<td style="text-align:right;">
0.089227
</td>
<td style="text-align:right;">
-0.009244
</td>
<td style="text-align:right;">
-0.012742
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Subj
</td>
<td style="text-align:right;">
-0.015302
</td>
<td style="text-align:right;">
0.174223
</td>
<td style="text-align:right;">
0.008649
</td>
<td style="text-align:right;">
0.114947
</td>
<td style="text-align:right;">
0.113228
</td>
<td style="text-align:right;">
-0.009754
</td>
<td style="text-align:right;">
0.042410
</td>
<td style="text-align:right;">
0.068725
</td>
<td style="text-align:right;">
0.141238
</td>
<td style="text-align:right;">
0.064004
</td>
<td style="text-align:right;">
-0.021358
</td>
<td style="text-align:right;">
0.061498
</td>
<td style="text-align:right;">
0.054498
</td>
<td style="text-align:right;">
0.009652
</td>
<td style="text-align:right;">
0.021083
</td>
<td style="text-align:right;">
0.004240
</td>
<td style="text-align:right;">
0.042448
</td>
<td style="text-align:right;">
0.050830
</td>
<td style="text-align:right;">
0.089336
</td>
<td style="text-align:right;">
0.039435
</td>
<td style="text-align:right;">
0.055510
</td>
<td style="text-align:right;">
0.052788
</td>
<td style="text-align:right;">
0.021474
</td>
<td style="text-align:right;">
-0.012861
</td>
<td style="text-align:right;">
-0.045286
</td>
<td style="text-align:right;">
0.051743
</td>
<td style="text-align:right;">
-0.019532
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.310279
</td>
<td style="text-align:right;">
0.294866
</td>
<td style="text-align:right;">
0.094474
</td>
<td style="text-align:right;">
0.200244
</td>
<td style="text-align:right;">
-0.071486
</td>
<td style="text-align:right;">
0.405215
</td>
<td style="text-align:right;">
0.010209
</td>
<td style="text-align:right;">
0.304105
</td>
<td style="text-align:right;">
-0.295899
</td>
<td style="text-align:right;">
-0.289741
</td>
<td style="text-align:right;">
-0.053422
</td>
<td style="text-align:right;">
0.113116
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pol
</td>
<td style="text-align:right;">
-0.038423
</td>
<td style="text-align:right;">
0.090778
</td>
<td style="text-align:right;">
-0.008961
</td>
<td style="text-align:right;">
0.015631
</td>
<td style="text-align:right;">
0.121085
</td>
<td style="text-align:right;">
-0.025945
</td>
<td style="text-align:right;">
0.007262
</td>
<td style="text-align:right;">
0.040273
</td>
<td style="text-align:right;">
0.116943
</td>
<td style="text-align:right;">
0.111318
</td>
<td style="text-align:right;">
0.041154
</td>
<td style="text-align:right;">
0.013305
</td>
<td style="text-align:right;">
0.024189
</td>
<td style="text-align:right;">
-0.032486
</td>
<td style="text-align:right;">
-0.043219
</td>
<td style="text-align:right;">
-0.067464
</td>
<td style="text-align:right;">
0.011730
</td>
<td style="text-align:right;">
0.012144
</td>
<td style="text-align:right;">
0.017802
</td>
<td style="text-align:right;">
-0.018593
</td>
<td style="text-align:right;">
-0.003695
</td>
<td style="text-align:right;">
-0.009756
</td>
<td style="text-align:right;">
0.121511
</td>
<td style="text-align:right;">
-0.032922
</td>
<td style="text-align:right;">
-0.118931
</td>
<td style="text-align:right;">
-0.051799
</td>
<td style="text-align:right;">
-0.026329
</td>
<td style="text-align:right;">
0.310279
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.598685
</td>
<td style="text-align:right;">
-0.506634
</td>
<td style="text-align:right;">
0.737398
</td>
<td style="text-align:right;">
-0.725062
</td>
<td style="text-align:right;">
0.490360
</td>
<td style="text-align:right;">
-0.101221
</td>
<td style="text-align:right;">
0.452755
</td>
<td style="text-align:right;">
0.273224
</td>
<td style="text-align:right;">
0.263612
</td>
<td style="text-align:right;">
0.025140
</td>
<td style="text-align:right;">
0.032152
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pos.Rate
</td>
<td style="text-align:right;">
-0.018033
</td>
<td style="text-align:right;">
0.178952
</td>
<td style="text-align:right;">
-0.054646
</td>
<td style="text-align:right;">
0.009303
</td>
<td style="text-align:right;">
0.132319
</td>
<td style="text-align:right;">
-0.025466
</td>
<td style="text-align:right;">
-0.032633
</td>
<td style="text-align:right;">
0.101639
</td>
<td style="text-align:right;">
0.133407
</td>
<td style="text-align:right;">
0.125501
</td>
<td style="text-align:right;">
0.048226
</td>
<td style="text-align:right;">
0.026836
</td>
<td style="text-align:right;">
0.046949
</td>
<td style="text-align:right;">
-0.013096
</td>
<td style="text-align:right;">
-0.053481
</td>
<td style="text-align:right;">
-0.072628
</td>
<td style="text-align:right;">
0.029399
</td>
<td style="text-align:right;">
0.033077
</td>
<td style="text-align:right;">
0.053369
</td>
<td style="text-align:right;">
-0.033334
</td>
<td style="text-align:right;">
0.013926
</td>
<td style="text-align:right;">
-0.007544
</td>
<td style="text-align:right;">
0.184908
</td>
<td style="text-align:right;">
-0.050156
</td>
<td style="text-align:right;">
-0.126306
</td>
<td style="text-align:right;">
-0.067472
</td>
<td style="text-align:right;">
-0.083953
</td>
<td style="text-align:right;">
0.294866
</td>
<td style="text-align:right;">
0.598685
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.031001
</td>
<td style="text-align:right;">
0.574601
</td>
<td style="text-align:right;">
-0.526395
</td>
<td style="text-align:right;">
0.140115
</td>
<td style="text-align:right;">
-0.346939
</td>
<td style="text-align:right;">
0.408904
</td>
<td style="text-align:right;">
-0.016214
</td>
<td style="text-align:right;">
-0.070908
</td>
<td style="text-align:right;">
0.052149
</td>
<td style="text-align:right;">
0.159400
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Neg.Rate
</td>
<td style="text-align:right;">
0.028051
</td>
<td style="text-align:right;">
0.090854
</td>
<td style="text-align:right;">
-0.041530
</td>
<td style="text-align:right;">
0.022318
</td>
<td style="text-align:right;">
0.002315
</td>
<td style="text-align:right;">
-0.007169
</td>
<td style="text-align:right;">
0.003014
</td>
<td style="text-align:right;">
0.018369
</td>
<td style="text-align:right;">
-0.041741
</td>
<td style="text-align:right;">
-0.028257
</td>
<td style="text-align:right;">
-0.030342
</td>
<td style="text-align:right;">
0.004767
</td>
<td style="text-align:right;">
0.005268
</td>
<td style="text-align:right;">
0.014274
</td>
<td style="text-align:right;">
0.016672
</td>
<td style="text-align:right;">
0.013668
</td>
<td style="text-align:right;">
0.017326
</td>
<td style="text-align:right;">
0.019672
</td>
<td style="text-align:right;">
0.038720
</td>
<td style="text-align:right;">
-0.016519
</td>
<td style="text-align:right;">
0.009229
</td>
<td style="text-align:right;">
-0.008681
</td>
<td style="text-align:right;">
-0.010781
</td>
<td style="text-align:right;">
0.010791
</td>
<td style="text-align:right;">
0.031634
</td>
<td style="text-align:right;">
0.012031
</td>
<td style="text-align:right;">
-0.021326
</td>
<td style="text-align:right;">
0.094474
</td>
<td style="text-align:right;">
-0.506634
</td>
<td style="text-align:right;">
-0.031001
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.694649
</td>
<td style="text-align:right;">
0.779172
</td>
<td style="text-align:right;">
0.059595
</td>
<td style="text-align:right;">
0.025567
</td>
<td style="text-align:right;">
0.029849
</td>
<td style="text-align:right;">
-0.274104
</td>
<td style="text-align:right;">
-0.437144
</td>
<td style="text-align:right;">
0.158664
</td>
<td style="text-align:right;">
0.059093
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Pos
</td>
<td style="text-align:right;">
-0.027998
</td>
<td style="text-align:right;">
0.057098
</td>
<td style="text-align:right;">
0.098463
</td>
<td style="text-align:right;">
0.128428
</td>
<td style="text-align:right;">
0.087561
</td>
<td style="text-align:right;">
0.011748
</td>
<td style="text-align:right;">
-0.002960
</td>
<td style="text-align:right;">
0.028553
</td>
<td style="text-align:right;">
0.308729
</td>
<td style="text-align:right;">
0.071565
</td>
<td style="text-align:right;">
0.037036
</td>
<td style="text-align:right;">
0.008048
</td>
<td style="text-align:right;">
0.021085
</td>
<td style="text-align:right;">
-0.024577
</td>
<td style="text-align:right;">
-0.032509
</td>
<td style="text-align:right;">
-0.060556
</td>
<td style="text-align:right;">
-0.007391
</td>
<td style="text-align:right;">
-0.018928
</td>
<td style="text-align:right;">
-0.026407
</td>
<td style="text-align:right;">
0.005658
</td>
<td style="text-align:right;">
0.008616
</td>
<td style="text-align:right;">
0.012976
</td>
<td style="text-align:right;">
0.122006
</td>
<td style="text-align:right;">
-0.019218
</td>
<td style="text-align:right;">
-0.081815
</td>
<td style="text-align:right;">
-0.097945
</td>
<td style="text-align:right;">
-0.033696
</td>
<td style="text-align:right;">
0.200244
</td>
<td style="text-align:right;">
0.737398
</td>
<td style="text-align:right;">
0.574601
</td>
<td style="text-align:right;">
-0.694649
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.901758
</td>
<td style="text-align:right;">
0.118368
</td>
<td style="text-align:right;">
-0.212830
</td>
<td style="text-align:right;">
0.282062
</td>
<td style="text-align:right;">
0.176695
</td>
<td style="text-align:right;">
0.263491
</td>
<td style="text-align:right;">
-0.095080
</td>
<td style="text-align:right;">
0.029738
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Neg
</td>
<td style="text-align:right;">
0.029181
</td>
<td style="text-align:right;">
-0.023737
</td>
<td style="text-align:right;">
0.045140
</td>
<td style="text-align:right;">
0.067471
</td>
<td style="text-align:right;">
-0.060199
</td>
<td style="text-align:right;">
0.014413
</td>
<td style="text-align:right;">
0.017603
</td>
<td style="text-align:right;">
-0.030782
</td>
<td style="text-align:right;">
0.003704
</td>
<td style="text-align:right;">
-0.103740
</td>
<td style="text-align:right;">
-0.046518
</td>
<td style="text-align:right;">
-0.007472
</td>
<td style="text-align:right;">
-0.016061
</td>
<td style="text-align:right;">
0.016915
</td>
<td style="text-align:right;">
0.035711
</td>
<td style="text-align:right;">
0.041416
</td>
<td style="text-align:right;">
-0.000731
</td>
<td style="text-align:right;">
-0.010139
</td>
<td style="text-align:right;">
-0.007812
</td>
<td style="text-align:right;">
-0.001880
</td>
<td style="text-align:right;">
-0.004850
</td>
<td style="text-align:right;">
-0.009279
</td>
<td style="text-align:right;">
-0.109006
</td>
<td style="text-align:right;">
0.020336
</td>
<td style="text-align:right;">
0.100592
</td>
<td style="text-align:right;">
0.024276
</td>
<td style="text-align:right;">
0.046588
</td>
<td style="text-align:right;">
-0.071486
</td>
<td style="text-align:right;">
-0.725062
</td>
<td style="text-align:right;">
-0.526395
</td>
<td style="text-align:right;">
0.779172
</td>
<td style="text-align:right;">
-0.901758
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.004852
</td>
<td style="text-align:right;">
0.261214
</td>
<td style="text-align:right;">
-0.191604
</td>
<td style="text-align:right;">
-0.250466
</td>
<td style="text-align:right;">
-0.328030
</td>
<td style="text-align:right;">
0.062325
</td>
<td style="text-align:right;">
-0.040230
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Pos.Pol
</td>
<td style="text-align:right;">
-0.026867
</td>
<td style="text-align:right;">
0.113263
</td>
<td style="text-align:right;">
0.052574
</td>
<td style="text-align:right;">
0.122726
</td>
<td style="text-align:right;">
0.118563
</td>
<td style="text-align:right;">
-0.014212
</td>
<td style="text-align:right;">
0.011254
</td>
<td style="text-align:right;">
0.048939
</td>
<td style="text-align:right;">
0.154391
</td>
<td style="text-align:right;">
0.043282
</td>
<td style="text-align:right;">
-0.016350
</td>
<td style="text-align:right;">
0.023270
</td>
<td style="text-align:right;">
0.024010
</td>
<td style="text-align:right;">
0.010245
</td>
<td style="text-align:right;">
0.006070
</td>
<td style="text-align:right;">
-0.010076
</td>
<td style="text-align:right;">
0.022243
</td>
<td style="text-align:right;">
0.018977
</td>
<td style="text-align:right;">
0.040306
</td>
<td style="text-align:right;">
0.000572
</td>
<td style="text-align:right;">
0.028924
</td>
<td style="text-align:right;">
0.013567
</td>
<td style="text-align:right;">
0.025086
</td>
<td style="text-align:right;">
-0.007544
</td>
<td style="text-align:right;">
-0.049298
</td>
<td style="text-align:right;">
-0.002969
</td>
<td style="text-align:right;">
0.007343
</td>
<td style="text-align:right;">
0.405215
</td>
<td style="text-align:right;">
0.490360
</td>
<td style="text-align:right;">
0.140115
</td>
<td style="text-align:right;">
0.059595
</td>
<td style="text-align:right;">
0.118368
</td>
<td style="text-align:right;">
0.004852
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.328305
</td>
<td style="text-align:right;">
0.554234
</td>
<td style="text-align:right;">
-0.112477
</td>
<td style="text-align:right;">
-0.122352
</td>
<td style="text-align:right;">
-0.028794
</td>
<td style="text-align:right;">
0.011277
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Pos.Pol
</td>
<td style="text-align:right;">
-0.003900
</td>
<td style="text-align:right;">
-0.332958
</td>
<td style="text-align:right;">
0.379052
</td>
<td style="text-align:right;">
0.300479
</td>
<td style="text-align:right;">
-0.222707
</td>
<td style="text-align:right;">
-0.041999
</td>
<td style="text-align:right;">
-0.064787
</td>
<td style="text-align:right;">
-0.028524
</td>
<td style="text-align:right;">
0.008935
</td>
<td style="text-align:right;">
-0.105683
</td>
<td style="text-align:right;">
0.005734
</td>
<td style="text-align:right;">
-0.021196
</td>
<td style="text-align:right;">
-0.036495
</td>
<td style="text-align:right;">
0.061832
</td>
<td style="text-align:right;">
-0.013973
</td>
<td style="text-align:right;">
0.049089
</td>
<td style="text-align:right;">
-0.006112
</td>
<td style="text-align:right;">
-0.021240
</td>
<td style="text-align:right;">
-0.026120
</td>
<td style="text-align:right;">
0.008484
</td>
<td style="text-align:right;">
-0.009294
</td>
<td style="text-align:right;">
-0.000161
</td>
<td style="text-align:right;">
-0.150526
</td>
<td style="text-align:right;">
0.035223
</td>
<td style="text-align:right;">
0.031953
</td>
<td style="text-align:right;">
0.033049
</td>
<td style="text-align:right;">
0.132995
</td>
<td style="text-align:right;">
0.010209
</td>
<td style="text-align:right;">
-0.101221
</td>
<td style="text-align:right;">
-0.346939
</td>
<td style="text-align:right;">
0.025567
</td>
<td style="text-align:right;">
-0.212830
</td>
<td style="text-align:right;">
0.261214
</td>
<td style="text-align:right;">
0.328305
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.234295
</td>
<td style="text-align:right;">
0.032976
</td>
<td style="text-align:right;">
0.172802
</td>
<td style="text-align:right;">
-0.138416
</td>
<td style="text-align:right;">
-0.039488
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Pos.Pol
</td>
<td style="text-align:right;">
-0.013722
</td>
<td style="text-align:right;">
0.463758
</td>
<td style="text-align:right;">
-0.322923
</td>
<td style="text-align:right;">
-0.184595
</td>
<td style="text-align:right;">
0.329171
</td>
<td style="text-align:right;">
0.025987
</td>
<td style="text-align:right;">
0.098895
</td>
<td style="text-align:right;">
0.085774
</td>
<td style="text-align:right;">
0.157859
</td>
<td style="text-align:right;">
0.141371
</td>
<td style="text-align:right;">
-0.039842
</td>
<td style="text-align:right;">
0.039729
</td>
<td style="text-align:right;">
0.045633
</td>
<td style="text-align:right;">
-0.031010
</td>
<td style="text-align:right;">
0.035732
</td>
<td style="text-align:right;">
-0.058639
</td>
<td style="text-align:right;">
0.015075
</td>
<td style="text-align:right;">
0.015943
</td>
<td style="text-align:right;">
0.035835
</td>
<td style="text-align:right;">
-0.027895
</td>
<td style="text-align:right;">
0.010761
</td>
<td style="text-align:right;">
-0.009345
</td>
<td style="text-align:right;">
0.134316
</td>
<td style="text-align:right;">
-0.042163
</td>
<td style="text-align:right;">
-0.075766
</td>
<td style="text-align:right;">
-0.044340
</td>
<td style="text-align:right;">
-0.071029
</td>
<td style="text-align:right;">
0.304105
</td>
<td style="text-align:right;">
0.452755
</td>
<td style="text-align:right;">
0.408904
</td>
<td style="text-align:right;">
0.029849
</td>
<td style="text-align:right;">
0.282062
</td>
<td style="text-align:right;">
-0.191604
</td>
<td style="text-align:right;">
0.554234
</td>
<td style="text-align:right;">
-0.234295
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.122486
</td>
<td style="text-align:right;">
-0.287987
</td>
<td style="text-align:right;">
0.128755
</td>
<td style="text-align:right;">
0.061725
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Neg.Pol
</td>
<td style="text-align:right;">
-0.017634
</td>
<td style="text-align:right;">
-0.143772
</td>
<td style="text-align:right;">
0.049102
</td>
<td style="text-align:right;">
-0.013454
</td>
<td style="text-align:right;">
-0.106653
</td>
<td style="text-align:right;">
-0.014408
</td>
<td style="text-align:right;">
-0.017271
</td>
<td style="text-align:right;">
-0.101376
</td>
<td style="text-align:right;">
-0.040404
</td>
<td style="text-align:right;">
-0.016445
</td>
<td style="text-align:right;">
0.064335
</td>
<td style="text-align:right;">
-0.050842
</td>
<td style="text-align:right;">
-0.040548
</td>
<td style="text-align:right;">
-0.034181
</td>
<td style="text-align:right;">
-0.065829
</td>
<td style="text-align:right;">
-0.081052
</td>
<td style="text-align:right;">
-0.034594
</td>
<td style="text-align:right;">
-0.072157
</td>
<td style="text-align:right;">
-0.102511
</td>
<td style="text-align:right;">
-0.078516
</td>
<td style="text-align:right;">
-0.087390
</td>
<td style="text-align:right;">
-0.089864
</td>
<td style="text-align:right;">
-0.017826
</td>
<td style="text-align:right;">
-0.006777
</td>
<td style="text-align:right;">
-0.006558
</td>
<td style="text-align:right;">
-0.065233
</td>
<td style="text-align:right;">
0.070322
</td>
<td style="text-align:right;">
-0.295899
</td>
<td style="text-align:right;">
0.273224
</td>
<td style="text-align:right;">
-0.016214
</td>
<td style="text-align:right;">
-0.274104
</td>
<td style="text-align:right;">
0.176695
</td>
<td style="text-align:right;">
-0.250466
</td>
<td style="text-align:right;">
-0.112477
</td>
<td style="text-align:right;">
0.032976
</td>
<td style="text-align:right;">
-0.122486
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.736673
</td>
<td style="text-align:right;">
0.537028
</td>
<td style="text-align:right;">
-0.058424
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Neg.Pol
</td>
<td style="text-align:right;">
-0.005758
</td>
<td style="text-align:right;">
-0.489611
</td>
<td style="text-align:right;">
0.373979
</td>
<td style="text-align:right;">
0.227442
</td>
<td style="text-align:right;">
-0.281240
</td>
<td style="text-align:right;">
-0.046358
</td>
<td style="text-align:right;">
-0.090883
</td>
<td style="text-align:right;">
-0.097916
</td>
<td style="text-align:right;">
0.015285
</td>
<td style="text-align:right;">
-0.068220
</td>
<td style="text-align:right;">
0.077298
</td>
<td style="text-align:right;">
-0.057745
</td>
<td style="text-align:right;">
-0.052646
</td>
<td style="text-align:right;">
-0.010573
</td>
<td style="text-align:right;">
-0.083724
</td>
<td style="text-align:right;">
-0.028571
</td>
<td style="text-align:right;">
-0.038540
</td>
<td style="text-align:right;">
-0.063959
</td>
<td style="text-align:right;">
-0.093344
</td>
<td style="text-align:right;">
-0.019040
</td>
<td style="text-align:right;">
-0.053293
</td>
<td style="text-align:right;">
-0.033144
</td>
<td style="text-align:right;">
-0.064247
</td>
<td style="text-align:right;">
0.034024
</td>
<td style="text-align:right;">
-0.021705
</td>
<td style="text-align:right;">
-0.020370
</td>
<td style="text-align:right;">
0.089227
</td>
<td style="text-align:right;">
-0.289741
</td>
<td style="text-align:right;">
0.263612
</td>
<td style="text-align:right;">
-0.070908
</td>
<td style="text-align:right;">
-0.437144
</td>
<td style="text-align:right;">
0.263491
</td>
<td style="text-align:right;">
-0.328030
</td>
<td style="text-align:right;">
-0.122352
</td>
<td style="text-align:right;">
0.172802
</td>
<td style="text-align:right;">
-0.287987
</td>
<td style="text-align:right;">
0.736673
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.000928
</td>
<td style="text-align:right;">
-0.051938
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Neg.Pol
</td>
<td style="text-align:right;">
-0.003502
</td>
<td style="text-align:right;">
0.276593
</td>
<td style="text-align:right;">
-0.318527
</td>
<td style="text-align:right;">
-0.246732
</td>
<td style="text-align:right;">
0.129478
</td>
<td style="text-align:right;">
0.008474
</td>
<td style="text-align:right;">
0.050848
</td>
<td style="text-align:right;">
-0.001326
</td>
<td style="text-align:right;">
-0.074783
</td>
<td style="text-align:right;">
0.048973
</td>
<td style="text-align:right;">
-0.013131
</td>
<td style="text-align:right;">
-0.000680
</td>
<td style="text-align:right;">
0.000405
</td>
<td style="text-align:right;">
-0.034984
</td>
<td style="text-align:right;">
0.015733
</td>
<td style="text-align:right;">
-0.063902
</td>
<td style="text-align:right;">
-0.000435
</td>
<td style="text-align:right;">
-0.021721
</td>
<td style="text-align:right;">
-0.029889
</td>
<td style="text-align:right;">
-0.109572
</td>
<td style="text-align:right;">
-0.065351
</td>
<td style="text-align:right;">
-0.101930
</td>
<td style="text-align:right;">
0.033366
</td>
<td style="text-align:right;">
-0.029982
</td>
<td style="text-align:right;">
0.010383
</td>
<td style="text-align:right;">
-0.038350
</td>
<td style="text-align:right;">
-0.009244
</td>
<td style="text-align:right;">
-0.053422
</td>
<td style="text-align:right;">
0.025140
</td>
<td style="text-align:right;">
0.052149
</td>
<td style="text-align:right;">
0.158664
</td>
<td style="text-align:right;">
-0.095080
</td>
<td style="text-align:right;">
0.062325
</td>
<td style="text-align:right;">
-0.028794
</td>
<td style="text-align:right;">
-0.138416
</td>
<td style="text-align:right;">
0.128755
</td>
<td style="text-align:right;">
0.537028
</td>
<td style="text-align:right;">
-0.000928
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.010786
</td>
</tr>
<tr>
<td style="text-align:left;">
Title.Subj
</td>
<td style="text-align:right;">
0.146469
</td>
<td style="text-align:right;">
0.035906
</td>
<td style="text-align:right;">
-0.002959
</td>
<td style="text-align:right;">
-0.008304
</td>
<td style="text-align:right;">
0.031429
</td>
<td style="text-align:right;">
-0.029379
</td>
<td style="text-align:right;">
0.010094
</td>
<td style="text-align:right;">
0.103662
</td>
<td style="text-align:right;">
-0.033952
</td>
<td style="text-align:right;">
0.031114
</td>
<td style="text-align:right;">
0.008704
</td>
<td style="text-align:right;">
0.003850
</td>
<td style="text-align:right;">
-0.007586
</td>
<td style="text-align:right;">
0.032256
</td>
<td style="text-align:right;">
-0.014187
</td>
<td style="text-align:right;">
0.039557
</td>
<td style="text-align:right;">
0.031850
</td>
<td style="text-align:right;">
0.051564
</td>
<td style="text-align:right;">
0.072562
</td>
<td style="text-align:right;">
0.001408
</td>
<td style="text-align:right;">
0.050487
</td>
<td style="text-align:right;">
0.030205
</td>
<td style="text-align:right;">
-0.002494
</td>
<td style="text-align:right;">
-0.017964
</td>
<td style="text-align:right;">
-0.035788
</td>
<td style="text-align:right;">
0.087760
</td>
<td style="text-align:right;">
-0.012742
</td>
<td style="text-align:right;">
0.113116
</td>
<td style="text-align:right;">
0.032152
</td>
<td style="text-align:right;">
0.159400
</td>
<td style="text-align:right;">
0.059093
</td>
<td style="text-align:right;">
0.029738
</td>
<td style="text-align:right;">
-0.040230
</td>
<td style="text-align:right;">
0.011277
</td>
<td style="text-align:right;">
-0.039488
</td>
<td style="text-align:right;">
0.061725
</td>
<td style="text-align:right;">
-0.058424
</td>
<td style="text-align:right;">
-0.051938
</td>
<td style="text-align:right;">
0.010786
</td>
<td style="text-align:right;">
1.000000
</td>
</tr>
</tbody>
</table>

The above table gives the correlations between all variables in the
Business data set. This allows us to see which two variables have strong
correlation. If we have two variables with a high correlation, we might
want to remove one of them to avoid too much multicollinearity.

``` r
#Correlation graph for lifestyle_train
correlation_graph(data_channel_train)
```

    ##                 Var1                Var2     value
    ## 82         n.Content         Rate.Unique -0.721974
    ## 122        n.Content Rate.Unique.Nonstop -0.557612
    ## 123      Rate.Unique Rate.Unique.Nonstop  0.905575
    ## 162        n.Content             n.Links  0.578959
    ## 492    Max.Worst.Key       Avg.Worst.Key  0.935909
    ## 571    Min.Worst.Key        Max.Best.Key -0.859758
    ## 611    Min.Worst.Key        Avg.Best.Key -0.646789
    ## 615     Max.Best.Key        Avg.Best.Key  0.670122
    ## 738      Avg.Max.Key         Avg.Avg.Key  0.876213
    ## 860          Min.Ref             Avg.Ref  0.826240
    ## 861          Max.Ref             Avg.Ref  0.869974
    ## 1063          LDA_00              LDA_04 -0.640016
    ## 1189      Global.Pol     Global.Pos.Rate  0.598685
    ## 1229      Global.Pol     Global.Neg.Rate -0.506634
    ## 1269      Global.Pol            Rate.Pos  0.737398
    ## 1270 Global.Pos.Rate            Rate.Pos  0.574601
    ## 1271 Global.Neg.Rate            Rate.Pos -0.694649
    ## 1309      Global.Pol            Rate.Neg -0.725062
    ## 1310 Global.Pos.Rate            Rate.Neg -0.526395
    ## 1311 Global.Neg.Rate            Rate.Neg  0.779172
    ## 1312        Rate.Pos            Rate.Neg -0.901758
    ## 1434     Avg.Pos.Pol         Max.Pos.Pol  0.554234
    ## 1517     Avg.Neg.Pol         Min.Neg.Pol  0.736673
    ## 1557     Avg.Neg.Pol         Max.Neg.Pol  0.537028

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/r%20params$DataChannel%20corr_graph-1.png)<!-- -->

Because the correlation table above is large, it can be difficult to
read. The correlation graph above gives a visual summary of the table.
Using the legend, we are able to see the correlations between variables,
how strong the correlation is, and in what direction.

``` r
ggplot(shareshigh, aes(x=Rate.Pos, y=Rate.Neg,
                       color=Days_of_Week)) +
    geom_point(size=2)
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/scatterplot-1.png)<!-- -->

Once seeing the correlation table and graph, it is possible to graph two
variables on a scatterplot. This provides a visual of the linear
relationship. A scatterplot of two variables in the Business dataset has
been created above.

``` r
## mean of shares 
mean(data_channel_train$shares)
```

    ## [1] 2889.68

``` r
## sd of shares 
sd(data_channel_train$shares)
```

    ## [1] 9916.94

``` r
## creates a new column that is if shares is higher than average or not 
shareshigh <- data_channel_train %>% select(shares) %>% mutate (shareshigh = (shares> mean(shares)))

## creates a contingency table of shareshigh and whether it is the weekend 
table(shareshigh$shareshigh, data_channel_train$Weekend)
```

    ##        
    ##            0    1
    ##   FALSE 3202  229
    ##   TRUE   767  182

These above contingency tables will look at the shareshigh factor which
says whether the number of shares is higher than the mean number of
shares or not and compares it to the weekend. Using these we can see if
the number of shares tends to be higher or not on the weekend.

``` r
## creates a new column that is if shares is higher than average or not 
shareshigh <- data_channel_train %>% mutate (shareshigh = (shares> mean(shares)))

## create a new column that combines Mon-Fri into weekdays
shareshigh <- mutate(shareshigh, 
                  Weekday = ifelse(Mon == 1 |
                                     Tues ==1 |
                                     Wed == 1 |
                                     Thurs == 1 |
                                     Fri == 1, 
                                    'Weekday', 'Weekend'))
shareshigh <- mutate(shareshigh, 
                  Days_of_Week = ifelse(Mon == 1 & 
                                Weekday == 'Weekday', 'Mon',
                              ifelse(Tues == 1  &
                                Weekday == "Weekday", 'Tues',
                              ifelse(Wed == 1 &
                                Weekday == "Weekday", 'Wed',
                              ifelse(Thurs ==1 &
                                Weekday == 'Weekday', 'Thurs',
                              ifelse(Fri == 1 & 
                                       Weekday == 'Weekday',
                                     'Fri', 'Weekend'))))))

shareshigh$Days_of_Week <- ordered(shareshigh$Days_of_Week, 
                                levels=c("Mon", "Tues",
                                         "Wed", "Thurs", 
                                         "Fri", "Weekend"))

## creates a contingency table of shareshigh and whether it is a weekday 
print(prop.table(table(shareshigh$Weekday,
                       shareshigh$shareshigh)))
```

    ##          
    ##               FALSE      TRUE
    ##   Weekday 0.7310502 0.1751142
    ##   Weekend 0.0522831 0.0415525

The contingency table above looks at the before-mentioned shareshigh
factor and compares it to the whether the day was a weekend or a
weekday. This allows us to see if shares tend to be higher on weekends
or weekdays. The frequencies are displayed as relative frequencies.

``` r
## creates  a contingency table of shareshigh and the day of the week
a <- prop.table(table(shareshigh$Days_of_Week,
                 shareshigh$shareshigh))
b <- as.data.frame(a)
print(a)
```

    ##          
    ##               FALSE      TRUE
    ##   Mon     0.1474886 0.0383562
    ##   Tues    0.1500000 0.0372146
    ##   Wed     0.1632420 0.0397260
    ##   Thurs   0.1639269 0.0340183
    ##   Fri     0.1063927 0.0257991
    ##   Weekend 0.0522831 0.0415525

After comparing shareshigh with whether or not the day was a weekend or
weekday, the above contingency table compares shareshigh for each
specific day of the week. Again, the frequencies are displayed as
relative frequencies.

``` r
ggplot(shareshigh, aes(x = Weekday, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Weekday or Weekend?') + 
  ylab('Relative Frequency')
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/weekday%20bar%20graph-1.png)<!-- -->

``` r
ggplot(shareshigh, aes(x = Days_of_Week, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Day of the Week') + 
  ylab('Relative Frequency')
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/day%20of%20the%20week%20graph-1.png)<!-- -->

The above bar graphs are a visual representation of the contingency
tables between weekends/weekdays and shareshigh and the days of the week
and shareshigh.. Graphs can improve the stakeholders ability to
interpret the results quickly.

``` r
a <- table(shareshigh$Days_of_Week)
# a <- prop.table(table(shareshigh$Days_of_Week,
#                  shareshigh$shareshigh))
b <- as.data.frame(a)
colnames(b) <- c('Day of Week', 'Freq')
b <- filter(b, Freq == max(b$Freq))
d <- as.character(b[1,1])
g <- mutate(shareshigh, 
                  Most_Freq = ifelse(Days_of_Week == d,
                                    'Most Freq Day',
                                    'Not Most Freq Day'
                                    ))
paste0(" For ", 
        params$DataChannel, " ", 
       d, " is the most frequent day of the week")
```

    ## [1] " For Business Wed is the most frequent day of the week"

``` r
table(shareshigh$shareshigh, g$Most_Freq)
```

    ##        
    ##         Most Freq Day Not Most Freq Day
    ##   FALSE           715              2716
    ##   TRUE            174               775

The above contingency table compares shareshigh to the Business day that
occurs most frequently. This allows us to see if the most frequent day
tends to have more shareshigh.

``` r
## creates plotting object of shares
a <- ggplot(data_channel_train, aes(x=shares))

## histogram of shares 
a+geom_histogram(color= "red", fill="blue")+ ggtitle("Shares histogram")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/shares%20histogram-1.png)<!-- -->

Above we can see the frequency distribution of shares of the Business
data channel. We should always see a long tail to the right because a
small number of articles will get a very high number of shares. But
looking at by looking at the distribution we can say how many shares
most of these articles got.

``` r
## creates plotting object with number of words in title and shares
b<- ggplot(data_channel_train, aes(x=n.Title, y=shares))

## creates a bar chart with number of words in title and shares 
b+ geom_col(fill="blue")+ ggtitle("Number of words in title vs shares") + labs(x="Number of words in title")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/col%20graph-1.png)<!-- -->

In the above graph we are looking at the number of shares based on how
many words are in the title of the article. if we see a large peak on at
the higher number of words it means for this data channel there were
more shares on longer titles, and if we see a peak at smaller number of
words then there were more shares on smaller titles.

``` r
## makes correlation of every variable with shares 
shares_correlations <- cor(data_channel_train)[1,] %>% sort() 

shares_correlations
```

    ##         Min.Neg.Pol         Avg.Neg.Pol         Rate.Unique           Avg.Words 
    ##        -0.064335187        -0.048572334        -0.022802407        -0.022673751 
    ## Rate.Unique.Nonstop         Min.Pos.Pol              LDA_04              LDA_02 
    ##        -0.022265000        -0.021948880        -0.020854418        -0.019373220 
    ##            Rate.Pos                 Fri                 Mon           Title.Pol 
    ##        -0.017554172        -0.017317938        -0.015388695        -0.008925290 
    ##          Global.Pol              LDA_01              LDA_00       Min.Worst.Key 
    ##        -0.007856739        -0.007259790        -0.006543621        -0.005623275 
    ##         Max.Neg.Pol                 Wed             n.Other        Rate.Nonstop 
    ##        -0.001301805        -0.000811028        -0.000396732         0.000337523 
    ##        Min.Best.Key               Thurs            Abs.Subj                 Sun 
    ##         0.000353236         0.002399421         0.004734399         0.006625151 
    ##       Avg.Worst.Key       Max.Worst.Key                Tues         Avg.Pos.Pol 
    ##         0.006829495         0.009663025         0.014152271         0.015595413 
    ##             n.Title            Rate.Neg     Global.Pos.Rate             Weekend 
    ##         0.017763299         0.018853086         0.019075222         0.019550993 
    ##                 Sat             Abs.Pol        Max.Best.Key          Title.Subj 
    ##         0.021741260         0.024839986         0.028434073         0.031355423 
    ##         Max.Pos.Pol        Avg.Best.Key             Min.Ref               n.Key 
    ##         0.038638504         0.038788874         0.040449173         0.040452844 
    ##         Avg.Min.Key     Global.Neg.Rate            n.Videos             Max.Ref 
    ##         0.040680151         0.040886098         0.041752600         0.044027104 
    ##             Avg.Ref         Global.Subj           n.Content         Avg.Max.Key 
    ##         0.052006624         0.056053050         0.058218023         0.064956208 
    ##             n.Links            n.Images              LDA_03         Avg.Avg.Key 
    ##         0.066685735         0.077490775         0.080123901         0.102569294 
    ##              shares 
    ##         1.000000000

``` r
## take the name of the highest correlated variable
highest_cor <-shares_correlations[52]  %>% names()

highest_cor
```

    ## [1] "Avg.Avg.Key"

``` r
## creats scatter plot looking at shares vs highest correlated variable
g <-ggplot(data_channel_train,  aes(y=shares, x= data_channel_train[[highest_cor]])) 


g+ geom_point(aes(color=as.factor(Weekend))) +geom_smooth(method = lm) + ggtitle(" Highest correlated variable with shares") + labs(x="Highest correlated variable vs shares", color="Weekend")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/graph%20of%20shares%20with%20highest%20correlated%20var-1.png)<!-- -->

The above graph looks at the relationship between shares and the
variable with the highest correlation for the Business data channel, and
colored based on whether or not it is the weekend. because this is the
most positively correlated variable we should always see an upward trend
but the more correlated they are the more the dots will fall onto the
line of best fit.

## Modeling

## Linear Regression

Linear regression is a tool with many applications available to data
scientists. In linear regression, a linear relationship between one
dependent variable and one or more independent variables is assumed. In
computerized linear regression, many linear regressions between the
response variable and the explanatory variable(s) are calculated. The
regression that is considered the “best fit” is the least squares
regression line. To determine the LSRL, the sum of the squared residuals
is calculated for each regression. The best model is the regression that
minimizes the sum of the squared residuals. Linear regression is used to
predict responses for explanatory variable(s); it is also used to
examine trends in the data.

### Linear regression 1

``` r
## linear regression model using all predictors 
set.seed(13)

linear_model_1 <- train( shares ~ ., 
                         data = data_channel_train,
                         method = "lm",
                         preProcess = c("center", "scale"),
                         trControl = trainControl(method = "cv", 
                                                  number = 5))

## prediction of test with model 
linear_model_1_pred <- predict(linear_model_1, newdata = dplyr::select(data_channel_test, -shares))

## storing error of model on test set 
linear_1_RMSE<- postResample(linear_model_1_pred, obs = data_channel_test$shares)
```

### Linear regression 2

``` r
#Removed rate.Nonstop because it was only 1 and removed the days of the week.
linear_model_2 <- train( shares ~. - Rate.Nonstop - Mon
                         - Tues - Wed - Thurs - Fri - Sat
                         - Sun - Weekend, 
                        data = data_channel_train,
                         method = "lm",
                         preProcess = c("center", 
                                        "scale"),
                         trControl = trainControl(
                           method= "cv", 
                           number = 5))
## prediction of test with model 
linear_model_2_pred <- predict(linear_model_2, newdata = dplyr::select(data_channel_test, -shares))

## storing error of model on test set 
linear_2_RMSE<- postResample(linear_model_2_pred, obs = data_channel_test$shares)
```

## Ensemble Models

### Random forest model

A random forest model is used in machine learning to generate
predictions or classifications. This is done through generating many
decision trees on many different samples and taking the average
(regression) or the majority vote (classification). Some of the benefits
to using random forest models are that over-fitting is minimized and the
model works with the presence of categorical and continuous variables.
With increased computer power and the increased knowledge in machine
learning, random forest models will continue to grow in popularity.

``` r
set.seed(10210526)
rfFit <- train(shares ~ ., 
        data = data_channel_train,
        method = "rf",
        trControl = trainControl(method = "cv",
                                        number = 5),
        preProcess = c("center", "scale"),
        tuneGrid = 
          data.frame(mtry = 1:sqrt(ncol(data_channel_train))))
rfFit_pred <- predict(rfFit, newdata = data_channel_test)
rfRMSE<- postResample(rfFit_pred, obs =
                            data_channel_test$shares)
```

### Boosted tree model

A decision tree makes a binary decision based on the value input. A
boosted tree model generates a predictive model based on an ensemble of
decision trees where better trees are generated based on the performance
of previous trees. Our boosted tree model can be tuned using four
different parameters: interaction.depth which defines the complexity of
the trees being built, n.trees which defines the number of trees built
(number of iterations), shrinkage which dictates the rate at which the
algorithm learns, and n.minobsinnode which dictates the number of
samples left to allow for a node to split.

``` r
## creates grid of possible tuning parameters 
gbm_grid <-  expand.grid(interaction.depth = c(1,4,7), 
  n.trees = c(1:20) , 
  shrinkage = 0.1,
  n.minobsinnode = c(10,20, 40))

## sets trainControl method 
fit_control <- trainControl(method = "repeatedcv",
                            number = 5,
                            repeats= 1)

set.seed(13)

## trains to find optimal tuning parameters except it is giving weird parameters 
gbm_tree_cv <- train(shares ~ . , data = data_channel_train,
                     method = "gbm",
                     preProcess = c("center", "scale"),
                     trControl = fit_control,
                     tuneGrid= gbm_grid,
                     verbose=FALSE)
## plot to visualize parameters 
plot(gbm_tree_cv)
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Business_files/figure-gfm/boosted%20tree%20tuning-1.png)<!-- -->

``` r
## test set prediction
boosted_tree_model_pred <- predict(gbm_tree_cv, newdata = dplyr::select(data_channel_test, -shares), n.trees = 7)

## stores results 
boosted_tree_RMSE <- postResample(boosted_tree_model_pred, obs = data_channel_test$shares)
```

## Comparison

``` r
## creates a data frame of the four models RMSE on the 
models_RMSE <- data.frame(linear_1_RMSE=linear_1_RMSE[1],
                         linear_2_RMSE=linear_2_RMSE[1], 
                         rfRMSE=rfRMSE[1],
                          boosted_tree_RMSE =
                           boosted_tree_RMSE[1] )

models_RMSE
```

    ##      linear_1_RMSE linear_2_RMSE  rfRMSE boosted_tree_RMSE
    ## RMSE       22786.1       22774.1 22718.3           22837.6

``` r
## gets the name of the column with the smallest rmse 
smallest_RMSE<-colnames(models_RMSE)[apply(models_RMSE,1,which.min)]

## declares the model with smallest RSME the winner 
paste0(" For ", 
        params$DataChannel, " ", 
       smallest_RMSE, " is the winner")
```

    ## [1] " For Business rfRMSE is the winner"

## Automation

This is the code used to automate the rendering of each document based
on the parameter of data_channel_is designated in the YAML.

``` r
## creates a list of all 6 desired params from online
data_channel_is <- c("Lifestyle", "Entertainment", "Business", "Social.Media", "Tech", "World")

## creates the output file name 
output_file <- paste0(data_channel_is, ".md")

#create a list for each channel with just the channel name parameter
params = lapply(data_channel_is, FUN = function(x){list(DataChannel = x)})

#put into a data frame
reports <- tibble(output_file, params)

## renders with params to all based on rows in reports
apply(reports, MARGIN=1, FUN = function(x){
## change first path to wherever yours is and output_dir to whatever folder you want it to output to   
rmarkdown::render("C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/_Rmd/ST_558_Project_2.Rmd", 
                  output_format = "github_document", 
                  output_dir = ".", 
                  output_file = x[[1]], 
                  params = x[[2]],
                  runtime = "static"
    )
  }
)
```
