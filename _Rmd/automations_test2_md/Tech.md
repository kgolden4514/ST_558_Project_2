Project 2
================
Kristina Golden and Demetrios Samaras
2023-07-02

# Tech

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

In this report we will be looking at the Tech data channel of the online
news popularity data set. This data set looks at a wide range of
variables from 39644 different news articles. The response variable that
we will be focusing on is **shares**. The purpose of this analysis is to
try to predict how many shares a Tech article will get based on the
values of those other variables. We will be modeling shares using two
different linear regression models and two ensemble tree based models.

## Read in the Data

``` r
setwd("C:/Documents/Github/ST_558_Project_2")
#setwd("C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2")
 

online <- read.csv('OnlineNewsPopularity.csv')
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

## Tech EDA

### Tech

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

## Tech Summarizations

``` r
#Shares table for data_channel_train
summary_table(data_channel_train)
```

    ##            Shares
    ## Minimum     36.00
    ## Q1        1100.00
    ## Median    1700.00
    ## Q3        3000.00
    ## Maximum 663600.00
    ## Mean      3120.75
    ## SD       10405.76

The above table displays the Tech 5-number summary for the shares. It
also includes the mean and standard deviation. Because the mean is
greater than the median, we suspect that the Tech shares distribution is
right skewed.

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
-0.010681
</td>
<td style="text-align:right;">
0.020863
</td>
<td style="text-align:right;">
0.018069
</td>
<td style="text-align:right;">
-0.112688
</td>
<td style="text-align:right;">
0.010702
</td>
<td style="text-align:right;">
-0.031941
</td>
<td style="text-align:right;">
0.022696
</td>
<td style="text-align:right;">
-0.092017
</td>
<td style="text-align:right;">
0.044066
</td>
<td style="text-align:right;">
-0.109409
</td>
<td style="text-align:right;">
-0.035043
</td>
<td style="text-align:right;">
-0.075983
</td>
<td style="text-align:right;">
0.004461
</td>
<td style="text-align:right;">
0.113788
</td>
<td style="text-align:right;">
0.155407
</td>
<td style="text-align:right;">
-0.021248
</td>
<td style="text-align:right;">
-0.022084
</td>
<td style="text-align:right;">
-0.043535
</td>
<td style="text-align:right;">
-0.009945
</td>
<td style="text-align:right;">
-0.017943
</td>
<td style="text-align:right;">
-0.016393
</td>
<td style="text-align:right;">
-0.012523
</td>
<td style="text-align:right;">
0.025524
</td>
<td style="text-align:right;">
0.003383
</td>
<td style="text-align:right;">
-0.028159
</td>
<td style="text-align:right;">
0.005048
</td>
<td style="text-align:right;">
-0.034017
</td>
<td style="text-align:right;">
-0.020023
</td>
<td style="text-align:right;">
-0.006067
</td>
<td style="text-align:right;">
-0.002741
</td>
<td style="text-align:right;">
-0.008585
</td>
<td style="text-align:right;">
0.006482
</td>
<td style="text-align:right;">
-0.011307
</td>
<td style="text-align:right;">
0.031014
</td>
<td style="text-align:right;">
-0.019551
</td>
<td style="text-align:right;">
-0.004415
</td>
<td style="text-align:right;">
-0.003300
</td>
<td style="text-align:right;">
-0.008559
</td>
<td style="text-align:right;">
0.081985
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Content
</td>
<td style="text-align:right;">
-0.010681
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.736854
</td>
<td style="text-align:right;">
-0.621351
</td>
<td style="text-align:right;">
0.533345
</td>
<td style="text-align:right;">
0.389460
</td>
<td style="text-align:right;">
0.510136
</td>
<td style="text-align:right;">
0.094924
</td>
<td style="text-align:right;">
-0.028232
</td>
<td style="text-align:right;">
0.185697
</td>
<td style="text-align:right;">
-0.021212
</td>
<td style="text-align:right;">
0.022715
</td>
<td style="text-align:right;">
0.015677
</td>
<td style="text-align:right;">
-0.027696
</td>
<td style="text-align:right;">
0.019173
</td>
<td style="text-align:right;">
-0.054076
</td>
<td style="text-align:right;">
-0.036673
</td>
<td style="text-align:right;">
0.037584
</td>
<td style="text-align:right;">
0.009076
</td>
<td style="text-align:right;">
-0.013242
</td>
<td style="text-align:right;">
0.010257
</td>
<td style="text-align:right;">
-0.008533
</td>
<td style="text-align:right;">
-0.033107
</td>
<td style="text-align:right;">
-0.032315
</td>
<td style="text-align:right;">
-0.024918
</td>
<td style="text-align:right;">
-0.037779
</td>
<td style="text-align:right;">
0.070479
</td>
<td style="text-align:right;">
0.115795
</td>
<td style="text-align:right;">
0.032428
</td>
<td style="text-align:right;">
0.145905
</td>
<td style="text-align:right;">
0.130388
</td>
<td style="text-align:right;">
-0.020440
</td>
<td style="text-align:right;">
0.044449
</td>
<td style="text-align:right;">
0.100165
</td>
<td style="text-align:right;">
-0.343912
</td>
<td style="text-align:right;">
0.446442
</td>
<td style="text-align:right;">
-0.123970
</td>
<td style="text-align:right;">
-0.473968
</td>
<td style="text-align:right;">
0.268230
</td>
<td style="text-align:right;">
0.070853
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique
</td>
<td style="text-align:right;">
0.020863
</td>
<td style="text-align:right;">
-0.736854
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.931240
</td>
<td style="text-align:right;">
-0.377302
</td>
<td style="text-align:right;">
-0.305425
</td>
<td style="text-align:right;">
-0.500205
</td>
<td style="text-align:right;">
-0.033029
</td>
<td style="text-align:right;">
0.287277
</td>
<td style="text-align:right;">
-0.156824
</td>
<td style="text-align:right;">
0.023934
</td>
<td style="text-align:right;">
-0.018990
</td>
<td style="text-align:right;">
-0.016300
</td>
<td style="text-align:right;">
0.024709
</td>
<td style="text-align:right;">
-0.022337
</td>
<td style="text-align:right;">
0.004398
</td>
<td style="text-align:right;">
0.023984
</td>
<td style="text-align:right;">
-0.021292
</td>
<td style="text-align:right;">
-0.005095
</td>
<td style="text-align:right;">
-0.001562
</td>
<td style="text-align:right;">
-0.022766
</td>
<td style="text-align:right;">
-0.009297
</td>
<td style="text-align:right;">
0.036366
</td>
<td style="text-align:right;">
0.028336
</td>
<td style="text-align:right;">
0.037358
</td>
<td style="text-align:right;">
0.051206
</td>
<td style="text-align:right;">
-0.085655
</td>
<td style="text-align:right;">
-0.004369
</td>
<td style="text-align:right;">
0.002724
</td>
<td style="text-align:right;">
-0.025735
</td>
<td style="text-align:right;">
-0.045077
</td>
<td style="text-align:right;">
0.094346
</td>
<td style="text-align:right;">
0.001683
</td>
<td style="text-align:right;">
0.032135
</td>
<td style="text-align:right;">
0.379897
</td>
<td style="text-align:right;">
-0.351171
</td>
<td style="text-align:right;">
0.084162
</td>
<td style="text-align:right;">
0.374274
</td>
<td style="text-align:right;">
-0.253881
</td>
<td style="text-align:right;">
-0.037337
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique.Nonstop
</td>
<td style="text-align:right;">
0.018069
</td>
<td style="text-align:right;">
-0.621351
</td>
<td style="text-align:right;">
0.931240
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.351707
</td>
<td style="text-align:right;">
-0.304651
</td>
<td style="text-align:right;">
-0.543322
</td>
<td style="text-align:right;">
-0.015764
</td>
<td style="text-align:right;">
0.289892
</td>
<td style="text-align:right;">
-0.125839
</td>
<td style="text-align:right;">
0.018724
</td>
<td style="text-align:right;">
-0.013247
</td>
<td style="text-align:right;">
-0.014150
</td>
<td style="text-align:right;">
0.026959
</td>
<td style="text-align:right;">
-0.012394
</td>
<td style="text-align:right;">
0.014418
</td>
<td style="text-align:right;">
0.025528
</td>
<td style="text-align:right;">
-0.000518
</td>
<td style="text-align:right;">
0.024932
</td>
<td style="text-align:right;">
-0.004503
</td>
<td style="text-align:right;">
-0.013675
</td>
<td style="text-align:right;">
-0.004840
</td>
<td style="text-align:right;">
0.045769
</td>
<td style="text-align:right;">
0.029285
</td>
<td style="text-align:right;">
0.056218
</td>
<td style="text-align:right;">
0.043829
</td>
<td style="text-align:right;">
-0.101582
</td>
<td style="text-align:right;">
0.047799
</td>
<td style="text-align:right;">
-0.019495
</td>
<td style="text-align:right;">
0.004670
</td>
<td style="text-align:right;">
0.022303
</td>
<td style="text-align:right;">
0.070030
</td>
<td style="text-align:right;">
0.050915
</td>
<td style="text-align:right;">
0.078856
</td>
<td style="text-align:right;">
0.302049
</td>
<td style="text-align:right;">
-0.249182
</td>
<td style="text-align:right;">
0.013878
</td>
<td style="text-align:right;">
0.245124
</td>
<td style="text-align:right;">
-0.212094
</td>
<td style="text-align:right;">
-0.034233
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Links
</td>
<td style="text-align:right;">
-0.112688
</td>
<td style="text-align:right;">
0.533345
</td>
<td style="text-align:right;">
-0.377302
</td>
<td style="text-align:right;">
-0.351707
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.552180
</td>
<td style="text-align:right;">
0.369141
</td>
<td style="text-align:right;">
0.019478
</td>
<td style="text-align:right;">
0.122072
</td>
<td style="text-align:right;">
0.202572
</td>
<td style="text-align:right;">
-0.039265
</td>
<td style="text-align:right;">
0.049497
</td>
<td style="text-align:right;">
0.033898
</td>
<td style="text-align:right;">
-0.003783
</td>
<td style="text-align:right;">
0.033692
</td>
<td style="text-align:right;">
-0.067651
</td>
<td style="text-align:right;">
0.054411
</td>
<td style="text-align:right;">
0.074432
</td>
<td style="text-align:right;">
0.098421
</td>
<td style="text-align:right;">
-0.019119
</td>
<td style="text-align:right;">
0.014187
</td>
<td style="text-align:right;">
-0.012571
</td>
<td style="text-align:right;">
0.015457
</td>
<td style="text-align:right;">
-0.045745
</td>
<td style="text-align:right;">
-0.009708
</td>
<td style="text-align:right;">
0.023908
</td>
<td style="text-align:right;">
0.010002
</td>
<td style="text-align:right;">
0.140561
</td>
<td style="text-align:right;">
0.101296
</td>
<td style="text-align:right;">
0.111096
</td>
<td style="text-align:right;">
0.014624
</td>
<td style="text-align:right;">
0.048397
</td>
<td style="text-align:right;">
-0.028287
</td>
<td style="text-align:right;">
0.104448
</td>
<td style="text-align:right;">
-0.222907
</td>
<td style="text-align:right;">
0.284077
</td>
<td style="text-align:right;">
-0.104733
</td>
<td style="text-align:right;">
-0.266370
</td>
<td style="text-align:right;">
0.130301
</td>
<td style="text-align:right;">
0.075272
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Other
</td>
<td style="text-align:right;">
0.010702
</td>
<td style="text-align:right;">
0.389460
</td>
<td style="text-align:right;">
-0.305425
</td>
<td style="text-align:right;">
-0.304651
</td>
<td style="text-align:right;">
0.552180
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.428100
</td>
<td style="text-align:right;">
0.027449
</td>
<td style="text-align:right;">
-0.004060
</td>
<td style="text-align:right;">
0.160384
</td>
<td style="text-align:right;">
-0.068303
</td>
<td style="text-align:right;">
0.006703
</td>
<td style="text-align:right;">
-0.016669
</td>
<td style="text-align:right;">
-0.008110
</td>
<td style="text-align:right;">
0.041912
</td>
<td style="text-align:right;">
-0.033018
</td>
<td style="text-align:right;">
0.028659
</td>
<td style="text-align:right;">
0.000321
</td>
<td style="text-align:right;">
-0.021334
</td>
<td style="text-align:right;">
-0.042511
</td>
<td style="text-align:right;">
0.031902
</td>
<td style="text-align:right;">
-0.021622
</td>
<td style="text-align:right;">
0.006619
</td>
<td style="text-align:right;">
-0.001434
</td>
<td style="text-align:right;">
-0.129582
</td>
<td style="text-align:right;">
-0.045688
</td>
<td style="text-align:right;">
0.112514
</td>
<td style="text-align:right;">
0.050755
</td>
<td style="text-align:right;">
0.078039
</td>
<td style="text-align:right;">
0.088937
</td>
<td style="text-align:right;">
-0.026839
</td>
<td style="text-align:right;">
0.070276
</td>
<td style="text-align:right;">
-0.055087
</td>
<td style="text-align:right;">
0.013365
</td>
<td style="text-align:right;">
-0.153362
</td>
<td style="text-align:right;">
0.191104
</td>
<td style="text-align:right;">
-0.039269
</td>
<td style="text-align:right;">
-0.145791
</td>
<td style="text-align:right;">
0.112511
</td>
<td style="text-align:right;">
0.040226
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Images
</td>
<td style="text-align:right;">
-0.031941
</td>
<td style="text-align:right;">
0.510136
</td>
<td style="text-align:right;">
-0.500205
</td>
<td style="text-align:right;">
-0.543322
</td>
<td style="text-align:right;">
0.369141
</td>
<td style="text-align:right;">
0.428100
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.038826
</td>
<td style="text-align:right;">
-0.036190
</td>
<td style="text-align:right;">
0.078207
</td>
<td style="text-align:right;">
0.022559
</td>
<td style="text-align:right;">
-0.007768
</td>
<td style="text-align:right;">
-0.002976
</td>
<td style="text-align:right;">
-0.029122
</td>
<td style="text-align:right;">
-0.040377
</td>
<td style="text-align:right;">
-0.099382
</td>
<td style="text-align:right;">
0.003230
</td>
<td style="text-align:right;">
-0.018549
</td>
<td style="text-align:right;">
-0.047604
</td>
<td style="text-align:right;">
-0.020155
</td>
<td style="text-align:right;">
-0.009375
</td>
<td style="text-align:right;">
-0.024943
</td>
<td style="text-align:right;">
-0.082116
</td>
<td style="text-align:right;">
0.009091
</td>
<td style="text-align:right;">
-0.117379
</td>
<td style="text-align:right;">
-0.046851
</td>
<td style="text-align:right;">
0.148486
</td>
<td style="text-align:right;">
0.032791
</td>
<td style="text-align:right;">
0.082565
</td>
<td style="text-align:right;">
0.074254
</td>
<td style="text-align:right;">
-0.004149
</td>
<td style="text-align:right;">
0.037782
</td>
<td style="text-align:right;">
-0.030731
</td>
<td style="text-align:right;">
0.023964
</td>
<td style="text-align:right;">
-0.175310
</td>
<td style="text-align:right;">
0.260223
</td>
<td style="text-align:right;">
-0.011064
</td>
<td style="text-align:right;">
-0.184136
</td>
<td style="text-align:right;">
0.164195
</td>
<td style="text-align:right;">
0.047447
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Videos
</td>
<td style="text-align:right;">
0.022696
</td>
<td style="text-align:right;">
0.094924
</td>
<td style="text-align:right;">
-0.033029
</td>
<td style="text-align:right;">
-0.015764
</td>
<td style="text-align:right;">
0.019478
</td>
<td style="text-align:right;">
0.027449
</td>
<td style="text-align:right;">
-0.038826
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.016551
</td>
<td style="text-align:right;">
0.035992
</td>
<td style="text-align:right;">
-0.019060
</td>
<td style="text-align:right;">
0.023195
</td>
<td style="text-align:right;">
-0.002021
</td>
<td style="text-align:right;">
0.009017
</td>
<td style="text-align:right;">
0.034142
</td>
<td style="text-align:right;">
0.053774
</td>
<td style="text-align:right;">
0.027401
</td>
<td style="text-align:right;">
0.038516
</td>
<td style="text-align:right;">
0.062751
</td>
<td style="text-align:right;">
0.007592
</td>
<td style="text-align:right;">
0.007495
</td>
<td style="text-align:right;">
0.009554
</td>
<td style="text-align:right;">
-0.025365
</td>
<td style="text-align:right;">
0.008061
</td>
<td style="text-align:right;">
-0.041614
</td>
<td style="text-align:right;">
0.130143
</td>
<td style="text-align:right;">
-0.020965
</td>
<td style="text-align:right;">
0.014727
</td>
<td style="text-align:right;">
0.002093
</td>
<td style="text-align:right;">
0.022733
</td>
<td style="text-align:right;">
0.024151
</td>
<td style="text-align:right;">
-0.003201
</td>
<td style="text-align:right;">
0.002064
</td>
<td style="text-align:right;">
0.018093
</td>
<td style="text-align:right;">
-0.015392
</td>
<td style="text-align:right;">
0.059395
</td>
<td style="text-align:right;">
-0.043776
</td>
<td style="text-align:right;">
-0.081687
</td>
<td style="text-align:right;">
0.030985
</td>
<td style="text-align:right;">
0.032172
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Words
</td>
<td style="text-align:right;">
-0.092017
</td>
<td style="text-align:right;">
-0.028232
</td>
<td style="text-align:right;">
0.287277
</td>
<td style="text-align:right;">
0.289892
</td>
<td style="text-align:right;">
0.122072
</td>
<td style="text-align:right;">
-0.004060
</td>
<td style="text-align:right;">
-0.036190
</td>
<td style="text-align:right;">
-0.016551
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.022623
</td>
<td style="text-align:right;">
-0.022940
</td>
<td style="text-align:right;">
0.042586
</td>
<td style="text-align:right;">
0.027638
</td>
<td style="text-align:right;">
0.004928
</td>
<td style="text-align:right;">
0.020274
</td>
<td style="text-align:right;">
0.018866
</td>
<td style="text-align:right;">
-0.001319
</td>
<td style="text-align:right;">
0.048546
</td>
<td style="text-align:right;">
0.047862
</td>
<td style="text-align:right;">
0.027287
</td>
<td style="text-align:right;">
0.033576
</td>
<td style="text-align:right;">
0.037155
</td>
<td style="text-align:right;">
0.041968
</td>
<td style="text-align:right;">
-0.033605
</td>
<td style="text-align:right;">
0.151427
</td>
<td style="text-align:right;">
-0.023256
</td>
<td style="text-align:right;">
-0.105515
</td>
<td style="text-align:right;">
0.165068
</td>
<td style="text-align:right;">
0.003316
</td>
<td style="text-align:right;">
0.036615
</td>
<td style="text-align:right;">
0.033703
</td>
<td style="text-align:right;">
0.175371
</td>
<td style="text-align:right;">
0.086712
</td>
<td style="text-align:right;">
0.122261
</td>
<td style="text-align:right;">
0.036223
</td>
<td style="text-align:right;">
0.049489
</td>
<td style="text-align:right;">
-0.048153
</td>
<td style="text-align:right;">
-0.009824
</td>
<td style="text-align:right;">
-0.041449
</td>
<td style="text-align:right;">
-0.008418
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Key
</td>
<td style="text-align:right;">
0.044066
</td>
<td style="text-align:right;">
0.185697
</td>
<td style="text-align:right;">
-0.156824
</td>
<td style="text-align:right;">
-0.125839
</td>
<td style="text-align:right;">
0.202572
</td>
<td style="text-align:right;">
0.160384
</td>
<td style="text-align:right;">
0.078207
</td>
<td style="text-align:right;">
0.035992
</td>
<td style="text-align:right;">
0.022623
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.035673
</td>
<td style="text-align:right;">
0.115058
</td>
<td style="text-align:right;">
0.107010
</td>
<td style="text-align:right;">
-0.209058
</td>
<td style="text-align:right;">
0.028833
</td>
<td style="text-align:right;">
-0.271026
</td>
<td style="text-align:right;">
-0.196179
</td>
<td style="text-align:right;">
0.129290
</td>
<td style="text-align:right;">
0.003194
</td>
<td style="text-align:right;">
-0.030955
</td>
<td style="text-align:right;">
-0.027337
</td>
<td style="text-align:right;">
-0.039484
</td>
<td style="text-align:right;">
-0.003820
</td>
<td style="text-align:right;">
-0.008728
</td>
<td style="text-align:right;">
0.025893
</td>
<td style="text-align:right;">
-0.024118
</td>
<td style="text-align:right;">
-0.000932
</td>
<td style="text-align:right;">
0.068916
</td>
<td style="text-align:right;">
0.089487
</td>
<td style="text-align:right;">
0.096588
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
0.047188
</td>
<td style="text-align:right;">
-0.052386
</td>
<td style="text-align:right;">
0.078432
</td>
<td style="text-align:right;">
-0.104877
</td>
<td style="text-align:right;">
0.160515
</td>
<td style="text-align:right;">
-0.027983
</td>
<td style="text-align:right;">
-0.091164
</td>
<td style="text-align:right;">
0.051760
</td>
<td style="text-align:right;">
0.021573
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Worst.Key
</td>
<td style="text-align:right;">
-0.109409
</td>
<td style="text-align:right;">
-0.021212
</td>
<td style="text-align:right;">
0.023934
</td>
<td style="text-align:right;">
0.018724
</td>
<td style="text-align:right;">
-0.039265
</td>
<td style="text-align:right;">
-0.068303
</td>
<td style="text-align:right;">
0.022559
</td>
<td style="text-align:right;">
-0.019060
</td>
<td style="text-align:right;">
-0.022940
</td>
<td style="text-align:right;">
-0.035673
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.034030
</td>
<td style="text-align:right;">
0.213244
</td>
<td style="text-align:right;">
-0.061806
</td>
<td style="text-align:right;">
-0.827056
</td>
<td style="text-align:right;">
-0.522155
</td>
<td style="text-align:right;">
-0.159098
</td>
<td style="text-align:right;">
-0.066346
</td>
<td style="text-align:right;">
-0.216674
</td>
<td style="text-align:right;">
-0.037507
</td>
<td style="text-align:right;">
-0.059236
</td>
<td style="text-align:right;">
-0.056960
</td>
<td style="text-align:right;">
0.000819
</td>
<td style="text-align:right;">
0.029561
</td>
<td style="text-align:right;">
-0.030787
</td>
<td style="text-align:right;">
0.016247
</td>
<td style="text-align:right;">
-0.000542
</td>
<td style="text-align:right;">
-0.006166
</td>
<td style="text-align:right;">
0.000513
</td>
<td style="text-align:right;">
0.017610
</td>
<td style="text-align:right;">
0.000500
</td>
<td style="text-align:right;">
-0.008158
</td>
<td style="text-align:right;">
-0.001898
</td>
<td style="text-align:right;">
-0.008972
</td>
<td style="text-align:right;">
0.003948
</td>
<td style="text-align:right;">
-0.020910
</td>
<td style="text-align:right;">
-0.000241
</td>
<td style="text-align:right;">
0.004766
</td>
<td style="text-align:right;">
-0.009220
</td>
<td style="text-align:right;">
-0.013426
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Worst.Key
</td>
<td style="text-align:right;">
-0.035043
</td>
<td style="text-align:right;">
0.022715
</td>
<td style="text-align:right;">
-0.018990
</td>
<td style="text-align:right;">
-0.013247
</td>
<td style="text-align:right;">
0.049497
</td>
<td style="text-align:right;">
0.006703
</td>
<td style="text-align:right;">
-0.007768
</td>
<td style="text-align:right;">
0.023195
</td>
<td style="text-align:right;">
0.042586
</td>
<td style="text-align:right;">
0.115058
</td>
<td style="text-align:right;">
0.034030
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.911331
</td>
<td style="text-align:right;">
-0.051549
</td>
<td style="text-align:right;">
-0.020241
</td>
<td style="text-align:right;">
-0.079819
</td>
<td style="text-align:right;">
0.011633
</td>
<td style="text-align:right;">
0.583181
</td>
<td style="text-align:right;">
0.360418
</td>
<td style="text-align:right;">
0.002291
</td>
<td style="text-align:right;">
-0.005278
</td>
<td style="text-align:right;">
-0.001127
</td>
<td style="text-align:right;">
0.009631
</td>
<td style="text-align:right;">
-0.014381
</td>
<td style="text-align:right;">
0.058397
</td>
<td style="text-align:right;">
0.012707
</td>
<td style="text-align:right;">
-0.046530
</td>
<td style="text-align:right;">
-0.005264
</td>
<td style="text-align:right;">
-0.004898
</td>
<td style="text-align:right;">
-0.003428
</td>
<td style="text-align:right;">
-0.003040
</td>
<td style="text-align:right;">
-0.005531
</td>
<td style="text-align:right;">
0.006333
</td>
<td style="text-align:right;">
0.017624
</td>
<td style="text-align:right;">
-0.003341
</td>
<td style="text-align:right;">
0.011604
</td>
<td style="text-align:right;">
-0.000499
</td>
<td style="text-align:right;">
-0.013645
</td>
<td style="text-align:right;">
0.016162
</td>
<td style="text-align:right;">
-0.011182
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Worst.Key
</td>
<td style="text-align:right;">
-0.075983
</td>
<td style="text-align:right;">
0.015677
</td>
<td style="text-align:right;">
-0.016300
</td>
<td style="text-align:right;">
-0.014150
</td>
<td style="text-align:right;">
0.033898
</td>
<td style="text-align:right;">
-0.016669
</td>
<td style="text-align:right;">
-0.002976
</td>
<td style="text-align:right;">
-0.002021
</td>
<td style="text-align:right;">
0.027638
</td>
<td style="text-align:right;">
0.107010
</td>
<td style="text-align:right;">
0.213244
</td>
<td style="text-align:right;">
0.911331
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.100190
</td>
<td style="text-align:right;">
-0.193625
</td>
<td style="text-align:right;">
-0.263188
</td>
<td style="text-align:right;">
-0.050652
</td>
<td style="text-align:right;">
0.498345
</td>
<td style="text-align:right;">
0.282108
</td>
<td style="text-align:right;">
-0.013040
</td>
<td style="text-align:right;">
-0.016443
</td>
<td style="text-align:right;">
-0.013665
</td>
<td style="text-align:right;">
0.007935
</td>
<td style="text-align:right;">
0.003186
</td>
<td style="text-align:right;">
0.078170
</td>
<td style="text-align:right;">
0.019587
</td>
<td style="text-align:right;">
-0.071936
</td>
<td style="text-align:right;">
-0.008288
</td>
<td style="text-align:right;">
0.000668
</td>
<td style="text-align:right;">
0.001641
</td>
<td style="text-align:right;">
-0.001828
</td>
<td style="text-align:right;">
-0.005893
</td>
<td style="text-align:right;">
0.004714
</td>
<td style="text-align:right;">
0.020911
</td>
<td style="text-align:right;">
0.005472
</td>
<td style="text-align:right;">
0.007870
</td>
<td style="text-align:right;">
0.005415
</td>
<td style="text-align:right;">
-0.010445
</td>
<td style="text-align:right;">
0.017649
</td>
<td style="text-align:right;">
-0.021515
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Best.Key
</td>
<td style="text-align:right;">
0.004461
</td>
<td style="text-align:right;">
-0.027696
</td>
<td style="text-align:right;">
0.024709
</td>
<td style="text-align:right;">
0.026959
</td>
<td style="text-align:right;">
-0.003783
</td>
<td style="text-align:right;">
-0.008110
</td>
<td style="text-align:right;">
-0.029122
</td>
<td style="text-align:right;">
0.009017
</td>
<td style="text-align:right;">
0.004928
</td>
<td style="text-align:right;">
-0.209058
</td>
<td style="text-align:right;">
-0.061806
</td>
<td style="text-align:right;">
-0.051549
</td>
<td style="text-align:right;">
-0.100190
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.070785
</td>
<td style="text-align:right;">
0.311002
</td>
<td style="text-align:right;">
0.323549
</td>
<td style="text-align:right;">
0.010238
</td>
<td style="text-align:right;">
0.169529
</td>
<td style="text-align:right;">
0.013836
</td>
<td style="text-align:right;">
0.014644
</td>
<td style="text-align:right;">
0.012049
</td>
<td style="text-align:right;">
0.025064
</td>
<td style="text-align:right;">
-0.018475
</td>
<td style="text-align:right;">
-0.028255
</td>
<td style="text-align:right;">
0.005794
</td>
<td style="text-align:right;">
0.013001
</td>
<td style="text-align:right;">
0.005738
</td>
<td style="text-align:right;">
-0.017408
</td>
<td style="text-align:right;">
-0.004170
</td>
<td style="text-align:right;">
0.021466
</td>
<td style="text-align:right;">
-0.019166
</td>
<td style="text-align:right;">
0.021447
</td>
<td style="text-align:right;">
-0.004772
</td>
<td style="text-align:right;">
-0.011120
</td>
<td style="text-align:right;">
-0.016569
</td>
<td style="text-align:right;">
-0.022996
</td>
<td style="text-align:right;">
-0.011956
</td>
<td style="text-align:right;">
-0.006960
</td>
<td style="text-align:right;">
0.013947
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Best.Key
</td>
<td style="text-align:right;">
0.113788
</td>
<td style="text-align:right;">
0.019173
</td>
<td style="text-align:right;">
-0.022337
</td>
<td style="text-align:right;">
-0.012394
</td>
<td style="text-align:right;">
0.033692
</td>
<td style="text-align:right;">
0.041912
</td>
<td style="text-align:right;">
-0.040377
</td>
<td style="text-align:right;">
0.034142
</td>
<td style="text-align:right;">
0.020274
</td>
<td style="text-align:right;">
0.028833
</td>
<td style="text-align:right;">
-0.827056
</td>
<td style="text-align:right;">
-0.020241
</td>
<td style="text-align:right;">
-0.193625
</td>
<td style="text-align:right;">
0.070785
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.601166
</td>
<td style="text-align:right;">
0.184191
</td>
<td style="text-align:right;">
0.096944
</td>
<td style="text-align:right;">
0.309821
</td>
<td style="text-align:right;">
0.044155
</td>
<td style="text-align:right;">
0.063617
</td>
<td style="text-align:right;">
0.064303
</td>
<td style="text-align:right;">
-0.001686
</td>
<td style="text-align:right;">
-0.042521
</td>
<td style="text-align:right;">
0.017960
</td>
<td style="text-align:right;">
-0.004229
</td>
<td style="text-align:right;">
0.011143
</td>
<td style="text-align:right;">
0.000121
</td>
<td style="text-align:right;">
-0.023697
</td>
<td style="text-align:right;">
-0.042846
</td>
<td style="text-align:right;">
-0.002650
</td>
<td style="text-align:right;">
-0.001950
</td>
<td style="text-align:right;">
0.011560
</td>
<td style="text-align:right;">
-0.002250
</td>
<td style="text-align:right;">
-0.006853
</td>
<td style="text-align:right;">
0.018191
</td>
<td style="text-align:right;">
-0.001059
</td>
<td style="text-align:right;">
-0.007582
</td>
<td style="text-align:right;">
0.010293
</td>
<td style="text-align:right;">
0.011750
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Best.Key
</td>
<td style="text-align:right;">
0.155407
</td>
<td style="text-align:right;">
-0.054076
</td>
<td style="text-align:right;">
0.004398
</td>
<td style="text-align:right;">
0.014418
</td>
<td style="text-align:right;">
-0.067651
</td>
<td style="text-align:right;">
-0.033018
</td>
<td style="text-align:right;">
-0.099382
</td>
<td style="text-align:right;">
0.053774
</td>
<td style="text-align:right;">
0.018866
</td>
<td style="text-align:right;">
-0.271026
</td>
<td style="text-align:right;">
-0.522155
</td>
<td style="text-align:right;">
-0.079819
</td>
<td style="text-align:right;">
-0.263188
</td>
<td style="text-align:right;">
0.311002
</td>
<td style="text-align:right;">
0.601166
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.351223
</td>
<td style="text-align:right;">
0.082117
</td>
<td style="text-align:right;">
0.362124
</td>
<td style="text-align:right;">
0.081673
</td>
<td style="text-align:right;">
0.128067
</td>
<td style="text-align:right;">
0.126077
</td>
<td style="text-align:right;">
0.039428
</td>
<td style="text-align:right;">
-0.071614
</td>
<td style="text-align:right;">
-0.003783
</td>
<td style="text-align:right;">
0.018956
</td>
<td style="text-align:right;">
0.007576
</td>
<td style="text-align:right;">
-0.010389
</td>
<td style="text-align:right;">
-0.075547
</td>
<td style="text-align:right;">
-0.102996
</td>
<td style="text-align:right;">
0.000358
</td>
<td style="text-align:right;">
-0.045940
</td>
<td style="text-align:right;">
0.049614
</td>
<td style="text-align:right;">
-0.048542
</td>
<td style="text-align:right;">
0.004307
</td>
<td style="text-align:right;">
-0.067712
</td>
<td style="text-align:right;">
-0.009871
</td>
<td style="text-align:right;">
0.006829
</td>
<td style="text-align:right;">
-0.012098
</td>
<td style="text-align:right;">
0.016208
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Min.Key
</td>
<td style="text-align:right;">
-0.021248
</td>
<td style="text-align:right;">
-0.036673
</td>
<td style="text-align:right;">
0.023984
</td>
<td style="text-align:right;">
0.025528
</td>
<td style="text-align:right;">
0.054411
</td>
<td style="text-align:right;">
0.028659
</td>
<td style="text-align:right;">
0.003230
</td>
<td style="text-align:right;">
0.027401
</td>
<td style="text-align:right;">
-0.001319
</td>
<td style="text-align:right;">
-0.196179
</td>
<td style="text-align:right;">
-0.159098
</td>
<td style="text-align:right;">
0.011633
</td>
<td style="text-align:right;">
-0.050652
</td>
<td style="text-align:right;">
0.323549
</td>
<td style="text-align:right;">
0.184191
</td>
<td style="text-align:right;">
0.351223
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.094910
</td>
<td style="text-align:right;">
0.538544
</td>
<td style="text-align:right;">
0.042031
</td>
<td style="text-align:right;">
0.055705
</td>
<td style="text-align:right;">
0.057482
</td>
<td style="text-align:right;">
-0.004030
</td>
<td style="text-align:right;">
-0.079774
</td>
<td style="text-align:right;">
-0.069304
</td>
<td style="text-align:right;">
-0.051079
</td>
<td style="text-align:right;">
0.116405
</td>
<td style="text-align:right;">
0.006243
</td>
<td style="text-align:right;">
-0.035949
</td>
<td style="text-align:right;">
-0.006784
</td>
<td style="text-align:right;">
0.009202
</td>
<td style="text-align:right;">
-0.016736
</td>
<td style="text-align:right;">
0.019176
</td>
<td style="text-align:right;">
-0.019596
</td>
<td style="text-align:right;">
-0.003660
</td>
<td style="text-align:right;">
-0.044383
</td>
<td style="text-align:right;">
-0.038974
</td>
<td style="text-align:right;">
-0.022180
</td>
<td style="text-align:right;">
-0.012939
</td>
<td style="text-align:right;">
0.028955
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Max.Key
</td>
<td style="text-align:right;">
-0.022084
</td>
<td style="text-align:right;">
0.037584
</td>
<td style="text-align:right;">
-0.021292
</td>
<td style="text-align:right;">
-0.000518
</td>
<td style="text-align:right;">
0.074432
</td>
<td style="text-align:right;">
0.000321
</td>
<td style="text-align:right;">
-0.018549
</td>
<td style="text-align:right;">
0.038516
</td>
<td style="text-align:right;">
0.048546
</td>
<td style="text-align:right;">
0.129290
</td>
<td style="text-align:right;">
-0.066346
</td>
<td style="text-align:right;">
0.583181
</td>
<td style="text-align:right;">
0.498345
</td>
<td style="text-align:right;">
0.010238
</td>
<td style="text-align:right;">
0.096944
</td>
<td style="text-align:right;">
0.082117
</td>
<td style="text-align:right;">
0.094910
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.757137
</td>
<td style="text-align:right;">
0.024983
</td>
<td style="text-align:right;">
0.036814
</td>
<td style="text-align:right;">
0.039514
</td>
<td style="text-align:right;">
0.055689
</td>
<td style="text-align:right;">
-0.032773
</td>
<td style="text-align:right;">
0.018135
</td>
<td style="text-align:right;">
0.121658
</td>
<td style="text-align:right;">
-0.085152
</td>
<td style="text-align:right;">
0.006212
</td>
<td style="text-align:right;">
-0.018400
</td>
<td style="text-align:right;">
0.002036
</td>
<td style="text-align:right;">
0.019421
</td>
<td style="text-align:right;">
-0.016246
</td>
<td style="text-align:right;">
0.017586
</td>
<td style="text-align:right;">
0.026173
</td>
<td style="text-align:right;">
-0.031498
</td>
<td style="text-align:right;">
0.032164
</td>
<td style="text-align:right;">
-0.037950
</td>
<td style="text-align:right;">
-0.055562
</td>
<td style="text-align:right;">
0.013107
</td>
<td style="text-align:right;">
0.027346
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Avg.Key
</td>
<td style="text-align:right;">
-0.043535
</td>
<td style="text-align:right;">
0.009076
</td>
<td style="text-align:right;">
-0.005095
</td>
<td style="text-align:right;">
0.024932
</td>
<td style="text-align:right;">
0.098421
</td>
<td style="text-align:right;">
-0.021334
</td>
<td style="text-align:right;">
-0.047604
</td>
<td style="text-align:right;">
0.062751
</td>
<td style="text-align:right;">
0.047862
</td>
<td style="text-align:right;">
0.003194
</td>
<td style="text-align:right;">
-0.216674
</td>
<td style="text-align:right;">
0.360418
</td>
<td style="text-align:right;">
0.282108
</td>
<td style="text-align:right;">
0.169529
</td>
<td style="text-align:right;">
0.309821
</td>
<td style="text-align:right;">
0.362124
</td>
<td style="text-align:right;">
0.538544
</td>
<td style="text-align:right;">
0.757137
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.064949
</td>
<td style="text-align:right;">
0.082590
</td>
<td style="text-align:right;">
0.091209
</td>
<td style="text-align:right;">
0.034076
</td>
<td style="text-align:right;">
-0.093635
</td>
<td style="text-align:right;">
-0.031504
</td>
<td style="text-align:right;">
0.090133
</td>
<td style="text-align:right;">
0.008152
</td>
<td style="text-align:right;">
0.023875
</td>
<td style="text-align:right;">
-0.039751
</td>
<td style="text-align:right;">
-0.027322
</td>
<td style="text-align:right;">
0.017508
</td>
<td style="text-align:right;">
-0.030729
</td>
<td style="text-align:right;">
0.028457
</td>
<td style="text-align:right;">
0.019467
</td>
<td style="text-align:right;">
-0.033194
</td>
<td style="text-align:right;">
0.004782
</td>
<td style="text-align:right;">
-0.060907
</td>
<td style="text-align:right;">
-0.064476
</td>
<td style="text-align:right;">
-0.000318
</td>
<td style="text-align:right;">
0.033253
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Ref
</td>
<td style="text-align:right;">
-0.009945
</td>
<td style="text-align:right;">
-0.013242
</td>
<td style="text-align:right;">
-0.001562
</td>
<td style="text-align:right;">
-0.004503
</td>
<td style="text-align:right;">
-0.019119
</td>
<td style="text-align:right;">
-0.042511
</td>
<td style="text-align:right;">
-0.020155
</td>
<td style="text-align:right;">
0.007592
</td>
<td style="text-align:right;">
0.027287
</td>
<td style="text-align:right;">
-0.030955
</td>
<td style="text-align:right;">
-0.037507
</td>
<td style="text-align:right;">
0.002291
</td>
<td style="text-align:right;">
-0.013040
</td>
<td style="text-align:right;">
0.013836
</td>
<td style="text-align:right;">
0.044155
</td>
<td style="text-align:right;">
0.081673
</td>
<td style="text-align:right;">
0.042031
</td>
<td style="text-align:right;">
0.024983
</td>
<td style="text-align:right;">
0.064949
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.463816
</td>
<td style="text-align:right;">
0.774731
</td>
<td style="text-align:right;">
-0.001286
</td>
<td style="text-align:right;">
-0.012615
</td>
<td style="text-align:right;">
0.006839
</td>
<td style="text-align:right;">
0.006645
</td>
<td style="text-align:right;">
-0.001071
</td>
<td style="text-align:right;">
0.002648
</td>
<td style="text-align:right;">
-0.013660
</td>
<td style="text-align:right;">
-0.026471
</td>
<td style="text-align:right;">
-0.005525
</td>
<td style="text-align:right;">
-0.002491
</td>
<td style="text-align:right;">
0.006052
</td>
<td style="text-align:right;">
-0.012239
</td>
<td style="text-align:right;">
0.009233
</td>
<td style="text-align:right;">
-0.041445
</td>
<td style="text-align:right;">
0.004928
</td>
<td style="text-align:right;">
0.012341
</td>
<td style="text-align:right;">
-0.001956
</td>
<td style="text-align:right;">
0.009398
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Ref
</td>
<td style="text-align:right;">
-0.017943
</td>
<td style="text-align:right;">
0.010257
</td>
<td style="text-align:right;">
-0.022766
</td>
<td style="text-align:right;">
-0.013675
</td>
<td style="text-align:right;">
0.014187
</td>
<td style="text-align:right;">
0.031902
</td>
<td style="text-align:right;">
-0.009375
</td>
<td style="text-align:right;">
0.007495
</td>
<td style="text-align:right;">
0.033576
</td>
<td style="text-align:right;">
-0.027337
</td>
<td style="text-align:right;">
-0.059236
</td>
<td style="text-align:right;">
-0.005278
</td>
<td style="text-align:right;">
-0.016443
</td>
<td style="text-align:right;">
0.014644
</td>
<td style="text-align:right;">
0.063617
</td>
<td style="text-align:right;">
0.128067
</td>
<td style="text-align:right;">
0.055705
</td>
<td style="text-align:right;">
0.036814
</td>
<td style="text-align:right;">
0.082590
</td>
<td style="text-align:right;">
0.463816
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.895775
</td>
<td style="text-align:right;">
0.006124
</td>
<td style="text-align:right;">
-0.025445
</td>
<td style="text-align:right;">
0.030537
</td>
<td style="text-align:right;">
-0.016581
</td>
<td style="text-align:right;">
-0.005050
</td>
<td style="text-align:right;">
0.025157
</td>
<td style="text-align:right;">
-0.049902
</td>
<td style="text-align:right;">
-0.037740
</td>
<td style="text-align:right;">
0.032217
</td>
<td style="text-align:right;">
-0.045426
</td>
<td style="text-align:right;">
0.051792
</td>
<td style="text-align:right;">
-0.003050
</td>
<td style="text-align:right;">
-0.013352
</td>
<td style="text-align:right;">
-0.017083
</td>
<td style="text-align:right;">
-0.036248
</td>
<td style="text-align:right;">
-0.028233
</td>
<td style="text-align:right;">
0.007525
</td>
<td style="text-align:right;">
-0.000682
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Ref
</td>
<td style="text-align:right;">
-0.016393
</td>
<td style="text-align:right;">
-0.008533
</td>
<td style="text-align:right;">
-0.009297
</td>
<td style="text-align:right;">
-0.004840
</td>
<td style="text-align:right;">
-0.012571
</td>
<td style="text-align:right;">
-0.021622
</td>
<td style="text-align:right;">
-0.024943
</td>
<td style="text-align:right;">
0.009554
</td>
<td style="text-align:right;">
0.037155
</td>
<td style="text-align:right;">
-0.039484
</td>
<td style="text-align:right;">
-0.056960
</td>
<td style="text-align:right;">
-0.001127
</td>
<td style="text-align:right;">
-0.013665
</td>
<td style="text-align:right;">
0.012049
</td>
<td style="text-align:right;">
0.064303
</td>
<td style="text-align:right;">
0.126077
</td>
<td style="text-align:right;">
0.057482
</td>
<td style="text-align:right;">
0.039514
</td>
<td style="text-align:right;">
0.091209
</td>
<td style="text-align:right;">
0.774731
</td>
<td style="text-align:right;">
0.895775
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.000335
</td>
<td style="text-align:right;">
-0.022795
</td>
<td style="text-align:right;">
0.029701
</td>
<td style="text-align:right;">
-0.005601
</td>
<td style="text-align:right;">
-0.007706
</td>
<td style="text-align:right;">
0.019480
</td>
<td style="text-align:right;">
-0.045151
</td>
<td style="text-align:right;">
-0.042977
</td>
<td style="text-align:right;">
0.021667
</td>
<td style="text-align:right;">
-0.035700
</td>
<td style="text-align:right;">
0.041695
</td>
<td style="text-align:right;">
-0.009307
</td>
<td style="text-align:right;">
0.000683
</td>
<td style="text-align:right;">
-0.040898
</td>
<td style="text-align:right;">
-0.022833
</td>
<td style="text-align:right;">
-0.011017
</td>
<td style="text-align:right;">
0.001194
</td>
<td style="text-align:right;">
0.002985
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_00
</td>
<td style="text-align:right;">
-0.012523
</td>
<td style="text-align:right;">
-0.033107
</td>
<td style="text-align:right;">
0.036366
</td>
<td style="text-align:right;">
0.045769
</td>
<td style="text-align:right;">
0.015457
</td>
<td style="text-align:right;">
0.006619
</td>
<td style="text-align:right;">
-0.082116
</td>
<td style="text-align:right;">
-0.025365
</td>
<td style="text-align:right;">
0.041968
</td>
<td style="text-align:right;">
-0.003820
</td>
<td style="text-align:right;">
0.000819
</td>
<td style="text-align:right;">
0.009631
</td>
<td style="text-align:right;">
0.007935
</td>
<td style="text-align:right;">
0.025064
</td>
<td style="text-align:right;">
-0.001686
</td>
<td style="text-align:right;">
0.039428
</td>
<td style="text-align:right;">
-0.004030
</td>
<td style="text-align:right;">
0.055689
</td>
<td style="text-align:right;">
0.034076
</td>
<td style="text-align:right;">
-0.001286
</td>
<td style="text-align:right;">
0.006124
</td>
<td style="text-align:right;">
0.000335
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.049476
</td>
<td style="text-align:right;">
-0.076789
</td>
<td style="text-align:right;">
-0.051402
</td>
<td style="text-align:right;">
-0.453580
</td>
<td style="text-align:right;">
-0.012006
</td>
<td style="text-align:right;">
0.018106
</td>
<td style="text-align:right;">
0.021025
</td>
<td style="text-align:right;">
-0.032737
</td>
<td style="text-align:right;">
0.028668
</td>
<td style="text-align:right;">
-0.035728
</td>
<td style="text-align:right;">
0.010494
</td>
<td style="text-align:right;">
-0.043632
</td>
<td style="text-align:right;">
0.002823
</td>
<td style="text-align:right;">
-0.031664
</td>
<td style="text-align:right;">
-0.005326
</td>
<td style="text-align:right;">
-0.035812
</td>
<td style="text-align:right;">
0.012399
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_01
</td>
<td style="text-align:right;">
0.025524
</td>
<td style="text-align:right;">
-0.032315
</td>
<td style="text-align:right;">
0.028336
</td>
<td style="text-align:right;">
0.029285
</td>
<td style="text-align:right;">
-0.045745
</td>
<td style="text-align:right;">
-0.001434
</td>
<td style="text-align:right;">
0.009091
</td>
<td style="text-align:right;">
0.008061
</td>
<td style="text-align:right;">
-0.033605
</td>
<td style="text-align:right;">
-0.008728
</td>
<td style="text-align:right;">
0.029561
</td>
<td style="text-align:right;">
-0.014381
</td>
<td style="text-align:right;">
0.003186
</td>
<td style="text-align:right;">
-0.018475
</td>
<td style="text-align:right;">
-0.042521
</td>
<td style="text-align:right;">
-0.071614
</td>
<td style="text-align:right;">
-0.079774
</td>
<td style="text-align:right;">
-0.032773
</td>
<td style="text-align:right;">
-0.093635
</td>
<td style="text-align:right;">
-0.012615
</td>
<td style="text-align:right;">
-0.025445
</td>
<td style="text-align:right;">
-0.022795
</td>
<td style="text-align:right;">
-0.049476
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.118598
</td>
<td style="text-align:right;">
-0.071708
</td>
<td style="text-align:right;">
-0.352037
</td>
<td style="text-align:right;">
-0.000837
</td>
<td style="text-align:right;">
-0.047841
</td>
<td style="text-align:right;">
-0.015518
</td>
<td style="text-align:right;">
0.067873
</td>
<td style="text-align:right;">
-0.055519
</td>
<td style="text-align:right;">
0.049976
</td>
<td style="text-align:right;">
-0.014087
</td>
<td style="text-align:right;">
0.022166
</td>
<td style="text-align:right;">
-0.025146
</td>
<td style="text-align:right;">
-0.035602
</td>
<td style="text-align:right;">
-0.020973
</td>
<td style="text-align:right;">
-0.005931
</td>
<td style="text-align:right;">
-0.002776
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_02
</td>
<td style="text-align:right;">
0.003383
</td>
<td style="text-align:right;">
-0.024918
</td>
<td style="text-align:right;">
0.037358
</td>
<td style="text-align:right;">
0.056218
</td>
<td style="text-align:right;">
-0.009708
</td>
<td style="text-align:right;">
-0.129582
</td>
<td style="text-align:right;">
-0.117379
</td>
<td style="text-align:right;">
-0.041614
</td>
<td style="text-align:right;">
0.151427
</td>
<td style="text-align:right;">
0.025893
</td>
<td style="text-align:right;">
-0.030787
</td>
<td style="text-align:right;">
0.058397
</td>
<td style="text-align:right;">
0.078170
</td>
<td style="text-align:right;">
-0.028255
</td>
<td style="text-align:right;">
0.017960
</td>
<td style="text-align:right;">
-0.003783
</td>
<td style="text-align:right;">
-0.069304
</td>
<td style="text-align:right;">
0.018135
</td>
<td style="text-align:right;">
-0.031504
</td>
<td style="text-align:right;">
0.006839
</td>
<td style="text-align:right;">
0.030537
</td>
<td style="text-align:right;">
0.029701
</td>
<td style="text-align:right;">
-0.076789
</td>
<td style="text-align:right;">
-0.118598
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.105832
</td>
<td style="text-align:right;">
-0.573329
</td>
<td style="text-align:right;">
-0.070822
</td>
<td style="text-align:right;">
-0.121927
</td>
<td style="text-align:right;">
-0.085890
</td>
<td style="text-align:right;">
0.055624
</td>
<td style="text-align:right;">
-0.100328
</td>
<td style="text-align:right;">
0.105360
</td>
<td style="text-align:right;">
-0.035439
</td>
<td style="text-align:right;">
-0.007634
</td>
<td style="text-align:right;">
-0.035123
</td>
<td style="text-align:right;">
-0.033809
</td>
<td style="text-align:right;">
-0.034566
</td>
<td style="text-align:right;">
-0.001580
</td>
<td style="text-align:right;">
-0.021073
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_03
</td>
<td style="text-align:right;">
-0.028159
</td>
<td style="text-align:right;">
-0.037779
</td>
<td style="text-align:right;">
0.051206
</td>
<td style="text-align:right;">
0.043829
</td>
<td style="text-align:right;">
0.023908
</td>
<td style="text-align:right;">
-0.045688
</td>
<td style="text-align:right;">
-0.046851
</td>
<td style="text-align:right;">
0.130143
</td>
<td style="text-align:right;">
-0.023256
</td>
<td style="text-align:right;">
-0.024118
</td>
<td style="text-align:right;">
0.016247
</td>
<td style="text-align:right;">
0.012707
</td>
<td style="text-align:right;">
0.019587
</td>
<td style="text-align:right;">
0.005794
</td>
<td style="text-align:right;">
-0.004229
</td>
<td style="text-align:right;">
0.018956
</td>
<td style="text-align:right;">
-0.051079
</td>
<td style="text-align:right;">
0.121658
</td>
<td style="text-align:right;">
0.090133
</td>
<td style="text-align:right;">
0.006645
</td>
<td style="text-align:right;">
-0.016581
</td>
<td style="text-align:right;">
-0.005601
</td>
<td style="text-align:right;">
-0.051402
</td>
<td style="text-align:right;">
-0.071708
</td>
<td style="text-align:right;">
-0.105832
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.329594
</td>
<td style="text-align:right;">
0.025007
</td>
<td style="text-align:right;">
0.004922
</td>
<td style="text-align:right;">
0.010386
</td>
<td style="text-align:right;">
0.031216
</td>
<td style="text-align:right;">
-0.024616
</td>
<td style="text-align:right;">
0.015004
</td>
<td style="text-align:right;">
0.049808
</td>
<td style="text-align:right;">
0.043092
</td>
<td style="text-align:right;">
-0.007258
</td>
<td style="text-align:right;">
-0.030185
</td>
<td style="text-align:right;">
-0.001365
</td>
<td style="text-align:right;">
-0.023732
</td>
<td style="text-align:right;">
0.062145
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_04
</td>
<td style="text-align:right;">
0.005048
</td>
<td style="text-align:right;">
0.070479
</td>
<td style="text-align:right;">
-0.085655
</td>
<td style="text-align:right;">
-0.101582
</td>
<td style="text-align:right;">
0.010002
</td>
<td style="text-align:right;">
0.112514
</td>
<td style="text-align:right;">
0.148486
</td>
<td style="text-align:right;">
-0.020965
</td>
<td style="text-align:right;">
-0.105515
</td>
<td style="text-align:right;">
-0.000932
</td>
<td style="text-align:right;">
-0.000542
</td>
<td style="text-align:right;">
-0.046530
</td>
<td style="text-align:right;">
-0.071936
</td>
<td style="text-align:right;">
0.013001
</td>
<td style="text-align:right;">
0.011143
</td>
<td style="text-align:right;">
0.007576
</td>
<td style="text-align:right;">
0.116405
</td>
<td style="text-align:right;">
-0.085152
</td>
<td style="text-align:right;">
0.008152
</td>
<td style="text-align:right;">
-0.001071
</td>
<td style="text-align:right;">
-0.005050
</td>
<td style="text-align:right;">
-0.007706
</td>
<td style="text-align:right;">
-0.453580
</td>
<td style="text-align:right;">
-0.352037
</td>
<td style="text-align:right;">
-0.573329
</td>
<td style="text-align:right;">
-0.329594
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.046709
</td>
<td style="text-align:right;">
0.099898
</td>
<td style="text-align:right;">
0.053416
</td>
<td style="text-align:right;">
-0.070664
</td>
<td style="text-align:right;">
0.096079
</td>
<td style="text-align:right;">
-0.088494
</td>
<td style="text-align:right;">
0.003427
</td>
<td style="text-align:right;">
-0.001469
</td>
<td style="text-align:right;">
0.039873
</td>
<td style="text-align:right;">
0.074189
</td>
<td style="text-align:right;">
0.039159
</td>
<td style="text-align:right;">
0.035272
</td>
<td style="text-align:right;">
-0.019512
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Subj
</td>
<td style="text-align:right;">
-0.034017
</td>
<td style="text-align:right;">
0.115795
</td>
<td style="text-align:right;">
-0.004369
</td>
<td style="text-align:right;">
0.047799
</td>
<td style="text-align:right;">
0.140561
</td>
<td style="text-align:right;">
0.050755
</td>
<td style="text-align:right;">
0.032791
</td>
<td style="text-align:right;">
0.014727
</td>
<td style="text-align:right;">
0.165068
</td>
<td style="text-align:right;">
0.068916
</td>
<td style="text-align:right;">
-0.006166
</td>
<td style="text-align:right;">
-0.005264
</td>
<td style="text-align:right;">
-0.008288
</td>
<td style="text-align:right;">
0.005738
</td>
<td style="text-align:right;">
0.000121
</td>
<td style="text-align:right;">
-0.010389
</td>
<td style="text-align:right;">
0.006243
</td>
<td style="text-align:right;">
0.006212
</td>
<td style="text-align:right;">
0.023875
</td>
<td style="text-align:right;">
0.002648
</td>
<td style="text-align:right;">
0.025157
</td>
<td style="text-align:right;">
0.019480
</td>
<td style="text-align:right;">
-0.012006
</td>
<td style="text-align:right;">
-0.000837
</td>
<td style="text-align:right;">
-0.070822
</td>
<td style="text-align:right;">
0.025007
</td>
<td style="text-align:right;">
0.046709
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.335887
</td>
<td style="text-align:right;">
0.310239
</td>
<td style="text-align:right;">
0.042212
</td>
<td style="text-align:right;">
0.222876
</td>
<td style="text-align:right;">
-0.111216
</td>
<td style="text-align:right;">
0.375429
</td>
<td style="text-align:right;">
0.010854
</td>
<td style="text-align:right;">
0.288603
</td>
<td style="text-align:right;">
-0.259318
</td>
<td style="text-align:right;">
-0.237877
</td>
<td style="text-align:right;">
-0.067084
</td>
<td style="text-align:right;">
0.117186
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pol
</td>
<td style="text-align:right;">
-0.020023
</td>
<td style="text-align:right;">
0.032428
</td>
<td style="text-align:right;">
0.002724
</td>
<td style="text-align:right;">
-0.019495
</td>
<td style="text-align:right;">
0.101296
</td>
<td style="text-align:right;">
0.078039
</td>
<td style="text-align:right;">
0.082565
</td>
<td style="text-align:right;">
0.002093
</td>
<td style="text-align:right;">
0.003316
</td>
<td style="text-align:right;">
0.089487
</td>
<td style="text-align:right;">
0.000513
</td>
<td style="text-align:right;">
-0.004898
</td>
<td style="text-align:right;">
0.000668
</td>
<td style="text-align:right;">
-0.017408
</td>
<td style="text-align:right;">
-0.023697
</td>
<td style="text-align:right;">
-0.075547
</td>
<td style="text-align:right;">
-0.035949
</td>
<td style="text-align:right;">
-0.018400
</td>
<td style="text-align:right;">
-0.039751
</td>
<td style="text-align:right;">
-0.013660
</td>
<td style="text-align:right;">
-0.049902
</td>
<td style="text-align:right;">
-0.045151
</td>
<td style="text-align:right;">
0.018106
</td>
<td style="text-align:right;">
-0.047841
</td>
<td style="text-align:right;">
-0.121927
</td>
<td style="text-align:right;">
0.004922
</td>
<td style="text-align:right;">
0.099898
</td>
<td style="text-align:right;">
0.335887
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.541237
</td>
<td style="text-align:right;">
-0.497028
</td>
<td style="text-align:right;">
0.719652
</td>
<td style="text-align:right;">
-0.709164
</td>
<td style="text-align:right;">
0.498797
</td>
<td style="text-align:right;">
0.026155
</td>
<td style="text-align:right;">
0.375277
</td>
<td style="text-align:right;">
0.272743
</td>
<td style="text-align:right;">
0.285815
</td>
<td style="text-align:right;">
0.015867
</td>
<td style="text-align:right;">
0.060525
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pos.Rate
</td>
<td style="text-align:right;">
-0.006067
</td>
<td style="text-align:right;">
0.145905
</td>
<td style="text-align:right;">
-0.025735
</td>
<td style="text-align:right;">
0.004670
</td>
<td style="text-align:right;">
0.111096
</td>
<td style="text-align:right;">
0.088937
</td>
<td style="text-align:right;">
0.074254
</td>
<td style="text-align:right;">
0.022733
</td>
<td style="text-align:right;">
0.036615
</td>
<td style="text-align:right;">
0.096588
</td>
<td style="text-align:right;">
0.017610
</td>
<td style="text-align:right;">
-0.003428
</td>
<td style="text-align:right;">
0.001641
</td>
<td style="text-align:right;">
-0.004170
</td>
<td style="text-align:right;">
-0.042846
</td>
<td style="text-align:right;">
-0.102996
</td>
<td style="text-align:right;">
-0.006784
</td>
<td style="text-align:right;">
0.002036
</td>
<td style="text-align:right;">
-0.027322
</td>
<td style="text-align:right;">
-0.026471
</td>
<td style="text-align:right;">
-0.037740
</td>
<td style="text-align:right;">
-0.042977
</td>
<td style="text-align:right;">
0.021025
</td>
<td style="text-align:right;">
-0.015518
</td>
<td style="text-align:right;">
-0.085890
</td>
<td style="text-align:right;">
0.010386
</td>
<td style="text-align:right;">
0.053416
</td>
<td style="text-align:right;">
0.310239
</td>
<td style="text-align:right;">
0.541237
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.038744
</td>
<td style="text-align:right;">
0.475350
</td>
<td style="text-align:right;">
-0.435202
</td>
<td style="text-align:right;">
0.150264
</td>
<td style="text-align:right;">
-0.250725
</td>
<td style="text-align:right;">
0.361291
</td>
<td style="text-align:right;">
-0.036749
</td>
<td style="text-align:right;">
-0.090701
</td>
<td style="text-align:right;">
0.035548
</td>
<td style="text-align:right;">
0.143771
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Neg.Rate
</td>
<td style="text-align:right;">
-0.002741
</td>
<td style="text-align:right;">
0.130388
</td>
<td style="text-align:right;">
-0.045077
</td>
<td style="text-align:right;">
0.022303
</td>
<td style="text-align:right;">
0.014624
</td>
<td style="text-align:right;">
-0.026839
</td>
<td style="text-align:right;">
-0.004149
</td>
<td style="text-align:right;">
0.024151
</td>
<td style="text-align:right;">
0.033703
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
0.000500
</td>
<td style="text-align:right;">
-0.003040
</td>
<td style="text-align:right;">
-0.001828
</td>
<td style="text-align:right;">
0.021466
</td>
<td style="text-align:right;">
-0.002650
</td>
<td style="text-align:right;">
0.000358
</td>
<td style="text-align:right;">
0.009202
</td>
<td style="text-align:right;">
0.019421
</td>
<td style="text-align:right;">
0.017508
</td>
<td style="text-align:right;">
-0.005525
</td>
<td style="text-align:right;">
0.032217
</td>
<td style="text-align:right;">
0.021667
</td>
<td style="text-align:right;">
-0.032737
</td>
<td style="text-align:right;">
0.067873
</td>
<td style="text-align:right;">
0.055624
</td>
<td style="text-align:right;">
0.031216
</td>
<td style="text-align:right;">
-0.070664
</td>
<td style="text-align:right;">
0.042212
</td>
<td style="text-align:right;">
-0.497028
</td>
<td style="text-align:right;">
0.038744
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.752032
</td>
<td style="text-align:right;">
0.815532
</td>
<td style="text-align:right;">
0.083220
</td>
<td style="text-align:right;">
-0.048386
</td>
<td style="text-align:right;">
0.106054
</td>
<td style="text-align:right;">
-0.250947
</td>
<td style="text-align:right;">
-0.443217
</td>
<td style="text-align:right;">
0.166968
</td>
<td style="text-align:right;">
0.060647
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Pos
</td>
<td style="text-align:right;">
-0.008585
</td>
<td style="text-align:right;">
-0.020440
</td>
<td style="text-align:right;">
0.094346
</td>
<td style="text-align:right;">
0.070030
</td>
<td style="text-align:right;">
0.048397
</td>
<td style="text-align:right;">
0.070276
</td>
<td style="text-align:right;">
0.037782
</td>
<td style="text-align:right;">
-0.003201
</td>
<td style="text-align:right;">
0.175371
</td>
<td style="text-align:right;">
0.047188
</td>
<td style="text-align:right;">
-0.008158
</td>
<td style="text-align:right;">
-0.005531
</td>
<td style="text-align:right;">
-0.005893
</td>
<td style="text-align:right;">
-0.019166
</td>
<td style="text-align:right;">
-0.001950
</td>
<td style="text-align:right;">
-0.045940
</td>
<td style="text-align:right;">
-0.016736
</td>
<td style="text-align:right;">
-0.016246
</td>
<td style="text-align:right;">
-0.030729
</td>
<td style="text-align:right;">
-0.002491
</td>
<td style="text-align:right;">
-0.045426
</td>
<td style="text-align:right;">
-0.035700
</td>
<td style="text-align:right;">
0.028668
</td>
<td style="text-align:right;">
-0.055519
</td>
<td style="text-align:right;">
-0.100328
</td>
<td style="text-align:right;">
-0.024616
</td>
<td style="text-align:right;">
0.096079
</td>
<td style="text-align:right;">
0.222876
</td>
<td style="text-align:right;">
0.719652
</td>
<td style="text-align:right;">
0.475350
</td>
<td style="text-align:right;">
-0.752032
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.925140
</td>
<td style="text-align:right;">
0.069288
</td>
<td style="text-align:right;">
-0.082471
</td>
<td style="text-align:right;">
0.144362
</td>
<td style="text-align:right;">
0.189783
</td>
<td style="text-align:right;">
0.312296
</td>
<td style="text-align:right;">
-0.115118
</td>
<td style="text-align:right;">
0.012797
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Neg
</td>
<td style="text-align:right;">
0.006482
</td>
<td style="text-align:right;">
0.044449
</td>
<td style="text-align:right;">
0.001683
</td>
<td style="text-align:right;">
0.050915
</td>
<td style="text-align:right;">
-0.028287
</td>
<td style="text-align:right;">
-0.055087
</td>
<td style="text-align:right;">
-0.030731
</td>
<td style="text-align:right;">
0.002064
</td>
<td style="text-align:right;">
0.086712
</td>
<td style="text-align:right;">
-0.052386
</td>
<td style="text-align:right;">
-0.001898
</td>
<td style="text-align:right;">
0.006333
</td>
<td style="text-align:right;">
0.004714
</td>
<td style="text-align:right;">
0.021447
</td>
<td style="text-align:right;">
0.011560
</td>
<td style="text-align:right;">
0.049614
</td>
<td style="text-align:right;">
0.019176
</td>
<td style="text-align:right;">
0.017586
</td>
<td style="text-align:right;">
0.028457
</td>
<td style="text-align:right;">
0.006052
</td>
<td style="text-align:right;">
0.051792
</td>
<td style="text-align:right;">
0.041695
</td>
<td style="text-align:right;">
-0.035728
</td>
<td style="text-align:right;">
0.049976
</td>
<td style="text-align:right;">
0.105360
</td>
<td style="text-align:right;">
0.015004
</td>
<td style="text-align:right;">
-0.088494
</td>
<td style="text-align:right;">
-0.111216
</td>
<td style="text-align:right;">
-0.709164
</td>
<td style="text-align:right;">
-0.435202
</td>
<td style="text-align:right;">
0.815532
</td>
<td style="text-align:right;">
-0.925140
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.024792
</td>
<td style="text-align:right;">
0.117389
</td>
<td style="text-align:right;">
-0.077869
</td>
<td style="text-align:right;">
-0.239860
</td>
<td style="text-align:right;">
-0.359214
</td>
<td style="text-align:right;">
0.095210
</td>
<td style="text-align:right;">
-0.010793
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Pos.Pol
</td>
<td style="text-align:right;">
-0.011307
</td>
<td style="text-align:right;">
0.100165
</td>
<td style="text-align:right;">
0.032135
</td>
<td style="text-align:right;">
0.078856
</td>
<td style="text-align:right;">
0.104448
</td>
<td style="text-align:right;">
0.013365
</td>
<td style="text-align:right;">
0.023964
</td>
<td style="text-align:right;">
0.018093
</td>
<td style="text-align:right;">
0.122261
</td>
<td style="text-align:right;">
0.078432
</td>
<td style="text-align:right;">
-0.008972
</td>
<td style="text-align:right;">
0.017624
</td>
<td style="text-align:right;">
0.020911
</td>
<td style="text-align:right;">
-0.004772
</td>
<td style="text-align:right;">
-0.002250
</td>
<td style="text-align:right;">
-0.048542
</td>
<td style="text-align:right;">
-0.019596
</td>
<td style="text-align:right;">
0.026173
</td>
<td style="text-align:right;">
0.019467
</td>
<td style="text-align:right;">
-0.012239
</td>
<td style="text-align:right;">
-0.003050
</td>
<td style="text-align:right;">
-0.009307
</td>
<td style="text-align:right;">
0.010494
</td>
<td style="text-align:right;">
-0.014087
</td>
<td style="text-align:right;">
-0.035439
</td>
<td style="text-align:right;">
0.049808
</td>
<td style="text-align:right;">
0.003427
</td>
<td style="text-align:right;">
0.375429
</td>
<td style="text-align:right;">
0.498797
</td>
<td style="text-align:right;">
0.150264
</td>
<td style="text-align:right;">
0.083220
</td>
<td style="text-align:right;">
0.069288
</td>
<td style="text-align:right;">
0.024792
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.352053
</td>
<td style="text-align:right;">
0.546440
</td>
<td style="text-align:right;">
-0.069202
</td>
<td style="text-align:right;">
-0.110195
</td>
<td style="text-align:right;">
0.019896
</td>
<td style="text-align:right;">
0.063395
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Pos.Pol
</td>
<td style="text-align:right;">
0.031014
</td>
<td style="text-align:right;">
-0.343912
</td>
<td style="text-align:right;">
0.379897
</td>
<td style="text-align:right;">
0.302049
</td>
<td style="text-align:right;">
-0.222907
</td>
<td style="text-align:right;">
-0.153362
</td>
<td style="text-align:right;">
-0.175310
</td>
<td style="text-align:right;">
-0.015392
</td>
<td style="text-align:right;">
0.036223
</td>
<td style="text-align:right;">
-0.104877
</td>
<td style="text-align:right;">
0.003948
</td>
<td style="text-align:right;">
-0.003341
</td>
<td style="text-align:right;">
0.005472
</td>
<td style="text-align:right;">
-0.011120
</td>
<td style="text-align:right;">
-0.006853
</td>
<td style="text-align:right;">
0.004307
</td>
<td style="text-align:right;">
-0.003660
</td>
<td style="text-align:right;">
-0.031498
</td>
<td style="text-align:right;">
-0.033194
</td>
<td style="text-align:right;">
0.009233
</td>
<td style="text-align:right;">
-0.013352
</td>
<td style="text-align:right;">
0.000683
</td>
<td style="text-align:right;">
-0.043632
</td>
<td style="text-align:right;">
0.022166
</td>
<td style="text-align:right;">
-0.007634
</td>
<td style="text-align:right;">
0.043092
</td>
<td style="text-align:right;">
-0.001469
</td>
<td style="text-align:right;">
0.010854
</td>
<td style="text-align:right;">
0.026155
</td>
<td style="text-align:right;">
-0.250725
</td>
<td style="text-align:right;">
-0.048386
</td>
<td style="text-align:right;">
-0.082471
</td>
<td style="text-align:right;">
0.117389
</td>
<td style="text-align:right;">
0.352053
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.195494
</td>
<td style="text-align:right;">
0.096045
</td>
<td style="text-align:right;">
0.230845
</td>
<td style="text-align:right;">
-0.112676
</td>
<td style="text-align:right;">
-0.018337
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Pos.Pol
</td>
<td style="text-align:right;">
-0.019551
</td>
<td style="text-align:right;">
0.446442
</td>
<td style="text-align:right;">
-0.351171
</td>
<td style="text-align:right;">
-0.249182
</td>
<td style="text-align:right;">
0.284077
</td>
<td style="text-align:right;">
0.191104
</td>
<td style="text-align:right;">
0.260223
</td>
<td style="text-align:right;">
0.059395
</td>
<td style="text-align:right;">
0.049489
</td>
<td style="text-align:right;">
0.160515
</td>
<td style="text-align:right;">
-0.020910
</td>
<td style="text-align:right;">
0.011604
</td>
<td style="text-align:right;">
0.007870
</td>
<td style="text-align:right;">
-0.016569
</td>
<td style="text-align:right;">
0.018191
</td>
<td style="text-align:right;">
-0.067712
</td>
<td style="text-align:right;">
-0.044383
</td>
<td style="text-align:right;">
0.032164
</td>
<td style="text-align:right;">
0.004782
</td>
<td style="text-align:right;">
-0.041445
</td>
<td style="text-align:right;">
-0.017083
</td>
<td style="text-align:right;">
-0.040898
</td>
<td style="text-align:right;">
0.002823
</td>
<td style="text-align:right;">
-0.025146
</td>
<td style="text-align:right;">
-0.035123
</td>
<td style="text-align:right;">
-0.007258
</td>
<td style="text-align:right;">
0.039873
</td>
<td style="text-align:right;">
0.288603
</td>
<td style="text-align:right;">
0.375277
</td>
<td style="text-align:right;">
0.361291
</td>
<td style="text-align:right;">
0.106054
</td>
<td style="text-align:right;">
0.144362
</td>
<td style="text-align:right;">
-0.077869
</td>
<td style="text-align:right;">
0.546440
</td>
<td style="text-align:right;">
-0.195494
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.143230
</td>
<td style="text-align:right;">
-0.320878
</td>
<td style="text-align:right;">
0.135332
</td>
<td style="text-align:right;">
0.078918
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Neg.Pol
</td>
<td style="text-align:right;">
-0.004415
</td>
<td style="text-align:right;">
-0.123970
</td>
<td style="text-align:right;">
0.084162
</td>
<td style="text-align:right;">
0.013878
</td>
<td style="text-align:right;">
-0.104733
</td>
<td style="text-align:right;">
-0.039269
</td>
<td style="text-align:right;">
-0.011064
</td>
<td style="text-align:right;">
-0.043776
</td>
<td style="text-align:right;">
-0.048153
</td>
<td style="text-align:right;">
-0.027983
</td>
<td style="text-align:right;">
-0.000241
</td>
<td style="text-align:right;">
-0.000499
</td>
<td style="text-align:right;">
0.005415
</td>
<td style="text-align:right;">
-0.022996
</td>
<td style="text-align:right;">
-0.001059
</td>
<td style="text-align:right;">
-0.009871
</td>
<td style="text-align:right;">
-0.038974
</td>
<td style="text-align:right;">
-0.037950
</td>
<td style="text-align:right;">
-0.060907
</td>
<td style="text-align:right;">
0.004928
</td>
<td style="text-align:right;">
-0.036248
</td>
<td style="text-align:right;">
-0.022833
</td>
<td style="text-align:right;">
-0.031664
</td>
<td style="text-align:right;">
-0.035602
</td>
<td style="text-align:right;">
-0.033809
</td>
<td style="text-align:right;">
-0.030185
</td>
<td style="text-align:right;">
0.074189
</td>
<td style="text-align:right;">
-0.259318
</td>
<td style="text-align:right;">
0.272743
</td>
<td style="text-align:right;">
-0.036749
</td>
<td style="text-align:right;">
-0.250947
</td>
<td style="text-align:right;">
0.189783
</td>
<td style="text-align:right;">
-0.239860
</td>
<td style="text-align:right;">
-0.069202
</td>
<td style="text-align:right;">
0.096045
</td>
<td style="text-align:right;">
-0.143230
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.728309
</td>
<td style="text-align:right;">
0.567244
</td>
<td style="text-align:right;">
-0.044526
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Neg.Pol
</td>
<td style="text-align:right;">
-0.003300
</td>
<td style="text-align:right;">
-0.473968
</td>
<td style="text-align:right;">
0.374274
</td>
<td style="text-align:right;">
0.245124
</td>
<td style="text-align:right;">
-0.266370
</td>
<td style="text-align:right;">
-0.145791
</td>
<td style="text-align:right;">
-0.184136
</td>
<td style="text-align:right;">
-0.081687
</td>
<td style="text-align:right;">
-0.009824
</td>
<td style="text-align:right;">
-0.091164
</td>
<td style="text-align:right;">
0.004766
</td>
<td style="text-align:right;">
-0.013645
</td>
<td style="text-align:right;">
-0.010445
</td>
<td style="text-align:right;">
-0.011956
</td>
<td style="text-align:right;">
-0.007582
</td>
<td style="text-align:right;">
0.006829
</td>
<td style="text-align:right;">
-0.022180
</td>
<td style="text-align:right;">
-0.055562
</td>
<td style="text-align:right;">
-0.064476
</td>
<td style="text-align:right;">
0.012341
</td>
<td style="text-align:right;">
-0.028233
</td>
<td style="text-align:right;">
-0.011017
</td>
<td style="text-align:right;">
-0.005326
</td>
<td style="text-align:right;">
-0.020973
</td>
<td style="text-align:right;">
-0.034566
</td>
<td style="text-align:right;">
-0.001365
</td>
<td style="text-align:right;">
0.039159
</td>
<td style="text-align:right;">
-0.237877
</td>
<td style="text-align:right;">
0.285815
</td>
<td style="text-align:right;">
-0.090701
</td>
<td style="text-align:right;">
-0.443217
</td>
<td style="text-align:right;">
0.312296
</td>
<td style="text-align:right;">
-0.359214
</td>
<td style="text-align:right;">
-0.110195
</td>
<td style="text-align:right;">
0.230845
</td>
<td style="text-align:right;">
-0.320878
</td>
<td style="text-align:right;">
0.728309
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.027050
</td>
<td style="text-align:right;">
-0.064561
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Neg.Pol
</td>
<td style="text-align:right;">
-0.008559
</td>
<td style="text-align:right;">
0.268230
</td>
<td style="text-align:right;">
-0.253881
</td>
<td style="text-align:right;">
-0.212094
</td>
<td style="text-align:right;">
0.130301
</td>
<td style="text-align:right;">
0.112511
</td>
<td style="text-align:right;">
0.164195
</td>
<td style="text-align:right;">
0.030985
</td>
<td style="text-align:right;">
-0.041449
</td>
<td style="text-align:right;">
0.051760
</td>
<td style="text-align:right;">
-0.009220
</td>
<td style="text-align:right;">
0.016162
</td>
<td style="text-align:right;">
0.017649
</td>
<td style="text-align:right;">
-0.006960
</td>
<td style="text-align:right;">
0.010293
</td>
<td style="text-align:right;">
-0.012098
</td>
<td style="text-align:right;">
-0.012939
</td>
<td style="text-align:right;">
0.013107
</td>
<td style="text-align:right;">
-0.000318
</td>
<td style="text-align:right;">
-0.001956
</td>
<td style="text-align:right;">
0.007525
</td>
<td style="text-align:right;">
0.001194
</td>
<td style="text-align:right;">
-0.035812
</td>
<td style="text-align:right;">
-0.005931
</td>
<td style="text-align:right;">
-0.001580
</td>
<td style="text-align:right;">
-0.023732
</td>
<td style="text-align:right;">
0.035272
</td>
<td style="text-align:right;">
-0.067084
</td>
<td style="text-align:right;">
0.015867
</td>
<td style="text-align:right;">
0.035548
</td>
<td style="text-align:right;">
0.166968
</td>
<td style="text-align:right;">
-0.115118
</td>
<td style="text-align:right;">
0.095210
</td>
<td style="text-align:right;">
0.019896
</td>
<td style="text-align:right;">
-0.112676
</td>
<td style="text-align:right;">
0.135332
</td>
<td style="text-align:right;">
0.567244
</td>
<td style="text-align:right;">
0.027050
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.011859
</td>
</tr>
<tr>
<td style="text-align:left;">
Title.Subj
</td>
<td style="text-align:right;">
0.081985
</td>
<td style="text-align:right;">
0.070853
</td>
<td style="text-align:right;">
-0.037337
</td>
<td style="text-align:right;">
-0.034233
</td>
<td style="text-align:right;">
0.075272
</td>
<td style="text-align:right;">
0.040226
</td>
<td style="text-align:right;">
0.047447
</td>
<td style="text-align:right;">
0.032172
</td>
<td style="text-align:right;">
-0.008418
</td>
<td style="text-align:right;">
0.021573
</td>
<td style="text-align:right;">
-0.013426
</td>
<td style="text-align:right;">
-0.011182
</td>
<td style="text-align:right;">
-0.021515
</td>
<td style="text-align:right;">
0.013947
</td>
<td style="text-align:right;">
0.011750
</td>
<td style="text-align:right;">
0.016208
</td>
<td style="text-align:right;">
0.028955
</td>
<td style="text-align:right;">
0.027346
</td>
<td style="text-align:right;">
0.033253
</td>
<td style="text-align:right;">
0.009398
</td>
<td style="text-align:right;">
-0.000682
</td>
<td style="text-align:right;">
0.002985
</td>
<td style="text-align:right;">
0.012399
</td>
<td style="text-align:right;">
-0.002776
</td>
<td style="text-align:right;">
-0.021073
</td>
<td style="text-align:right;">
0.062145
</td>
<td style="text-align:right;">
-0.019512
</td>
<td style="text-align:right;">
0.117186
</td>
<td style="text-align:right;">
0.060525
</td>
<td style="text-align:right;">
0.143771
</td>
<td style="text-align:right;">
0.060647
</td>
<td style="text-align:right;">
0.012797
</td>
<td style="text-align:right;">
-0.010793
</td>
<td style="text-align:right;">
0.063395
</td>
<td style="text-align:right;">
-0.018337
</td>
<td style="text-align:right;">
0.078918
</td>
<td style="text-align:right;">
-0.044526
</td>
<td style="text-align:right;">
-0.064561
</td>
<td style="text-align:right;">
0.011859
</td>
<td style="text-align:right;">
1.000000
</td>
</tr>
</tbody>
</table>

The above table gives the correlations between all variables in the Tech
data set. This allows us to see which two variables have strong
correlation. If we have two variables with a high correlation, we might
want to remove one of them to avoid too much multicollinearity.

``` r
#Correlation graph for lifestyle_train
correlation_graph(data_channel_train)
```

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/r%20params$DataChannel%20corr_graph-1.png)<!-- -->

Because the correlation table above is large, it can be difficult to
read. The correlation graph above gives a visual summary of the table.
Using the legend, we are able to see the correlations between variables,
how strong the correlation is, and in what direction.

``` r
ggplot(shareshigh, aes(x=Rate.Pos, y=Rate.Neg,
                       color=Days_of_Week)) +
    geom_point(size=2)
```

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/scatterplot-1.png)<!-- -->

Once seeing the correlation table and graph, it is possible to graph two
variables on a scatterplot. This provides a visual of the linear
relationship. A scatterplot of two variables in the Tech dataset has
been created above.

``` r
## mean of shares 
mean(data_channel_train$shares)
```

    ## [1] 3120.75

``` r
## sd of shares 
sd(data_channel_train$shares)
```

    ## [1] 10405.8

``` r
## creates a new column that is if shares is higher than average or not 
shareshigh <- data_channel_train %>% select(shares) %>% mutate (shareshigh = (shares> mean(shares)))

## creates a contingency table of shareshigh and whether it is the weekend 
table(shareshigh$shareshigh, data_channel_train$Weekend)
```

    ##        
    ##            0    1
    ##   FALSE 3507  421
    ##   TRUE  1006  208

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
    ##   Weekday 0.6820303 0.1956437
    ##   Weekend 0.0818748 0.0404512

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
    ##   Mon     0.1262155 0.0396733
    ##   Tues    0.1561649 0.0462855
    ##   Wed     0.1571373 0.0390898
    ##   Thurs   0.1398289 0.0375340
    ##   Fri     0.1026838 0.0330611
    ##   Weekend 0.0818748 0.0404512

After comparing shareshigh with whether or not the day was a weekend or
weekday, the above contingency table compares shareshigh for each
specific day of the week. Again, the frequencies are displayed as
relative frequencies.

``` r
ggplot(shareshigh, aes(x = Weekday, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Weekday or Weekend?') + 
  ylab('Relative Frequency')
```

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/weekday%20bar%20graph-1.png)<!-- -->

``` r
ggplot(shareshigh, aes(x = Days_of_Week, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Day of the Week') + 
  ylab('Relative Frequency')
```

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/day%20of%20the%20week%20graph-1.png)<!-- -->

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

    ## [1] " For Tech Tues is the most frequent day of the week"

``` r
table(shareshigh$shareshigh, g$Most_Freq)
```

    ##        
    ##         Most Freq Day Not Most Freq Day
    ##   FALSE           803              3125
    ##   TRUE            238               976

The above contingency table compares shareshigh to the Tech day that
occurs most frequently. This allows us to see if the most frequent day
tends to have more shareshigh.

``` r
## creates plotting object of shares
a <- ggplot(data_channel_train, aes(x=shares))

## histogram of shares 
a+geom_histogram(color= "red", fill="blue")+ ggtitle("Shares histogram")
```

    ## `stat_bin()` using `bins = 30`. Pick better value with
    ## `binwidth`.

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/shares%20histogram-1.png)<!-- -->

Above we can see the frequency distribution of shares of the Tech data
channel. We should always see a long tail to the right because a small
number of articles will get a very high number of shares. But looking at
by looking at the distribution we can say how many shares most of these
articles got.

``` r
## creates plotting object with number of words in title and shares
b<- ggplot(data_channel_train, aes(x=n.Title, y=shares))

## creates a bar chart with number of words in title and shares 
b+ geom_col(fill="blue")+ ggtitle("Number of words in title vs shares") + labs(x="Number of words in title")
```

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/col%20graph-1.png)<!-- -->

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

    ##         Rate.Unique Rate.Unique.Nonstop            Rate.Pos 
    ##        -0.055812087        -0.046838717        -0.033487909 
    ##         Min.Neg.Pol         Min.Pos.Pol         Avg.Neg.Pol 
    ##        -0.033281789        -0.031210450        -0.028750838 
    ##          Global.Pol     Global.Pos.Rate            Abs.Subj 
    ##        -0.028388031        -0.021736046        -0.021346460 
    ##        Avg.Best.Key              LDA_04               Thurs 
    ##        -0.016650264        -0.014746578        -0.011168178 
    ##                 Mon                Tues        Min.Best.Key 
    ##        -0.009710531        -0.007937566        -0.005189175 
    ##       Avg.Worst.Key        Rate.Nonstop              LDA_01 
    ##        -0.004008096        -0.003095709        -0.003008329 
    ##              LDA_02       Max.Worst.Key                 Fri 
    ##        -0.002173333        -0.001999427        -0.001763274 
    ##           Avg.Words         Avg.Pos.Pol             n.Title 
    ##        -0.000735722        -0.000464328         0.000855822 
    ##             n.Other        Max.Best.Key              LDA_03 
    ##         0.001112180         0.003155817         0.004425075 
    ##         Global.Subj                 Wed       Min.Worst.Key 
    ##         0.006644632         0.007236995         0.009051126 
    ##            n.Images             Min.Ref             Max.Ref 
    ##         0.009306616         0.011474401         0.011832179 
    ##                 Sat          Title.Subj             Avg.Ref 
    ##         0.013120552         0.013377380         0.015846177 
    ##             Abs.Pol         Avg.Min.Key           Title.Pol 
    ##         0.016207115         0.019143826         0.019234053 
    ##         Avg.Max.Key         Max.Pos.Pol               n.Key 
    ##         0.019397291         0.020149635         0.021593156 
    ##         Max.Neg.Pol                 Sun     Global.Neg.Rate 
    ##         0.022422659         0.024451099         0.025578125 
    ##             Weekend              LDA_00            n.Videos 
    ##         0.026849193         0.028201573         0.031519163 
    ##            Rate.Neg         Avg.Avg.Key           n.Content 
    ##         0.033542655         0.041690340         0.075426368 
    ##             n.Links              shares 
    ##         0.077726778         1.000000000

``` r
## take the name of the highest correlated variable
highest_cor <-shares_correlations[52]  %>% names()

highest_cor
```

    ## [1] "n.Links"

``` r
## creats scatter plot looking at shares vs highest correlated variable
g <-ggplot(data_channel_train,  aes(y=shares, x= data_channel_train[[highest_cor]])) 


g+ geom_point(aes(color=as.factor(Weekend))) +geom_smooth(method = lm) + ggtitle(" Highest correlated variable with shares") + labs(x="Highest correlated variable vs shares", color="Weekend")
```

    ## `geom_smooth()` using formula = 'y ~ x'

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/graph%20of%20shares%20with%20highest%20correlated%20var-1.png)<!-- -->

The above graph looks at the relationship between shares and the
variable with the highest correlation for the Tech data channel, and
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

![](C:/Documents/Github/ST_558_Project_2/_Rmd/automations_test2_md/Tech_files/figure-gfm/boosted%20tree%20tuning-1.png)<!-- -->

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

## gets the name of the column with the smallest rmse 
smallest_RMSE<-colnames(models_RMSE)[apply(models_RMSE,1,which.min)]

## declares the model with smallest RSME the winner 
paste0(" For ", 
        params$DataChannel, " ", 
       smallest_RMSE, " is the winner")
```

    ## [1] " For Tech rfRMSE is the winner"

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
rmarkdown::render('C:/Documents/Github/ST_558_Project_2/_Rmd/ST_558_Project_2.Rmd', output_dir = "./automations_test2_md", output_file = x[[1]], params = x[[2]]
    )
  }
)
```
