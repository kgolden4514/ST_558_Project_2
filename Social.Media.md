Project 2
================
Kristina Golden and Demetrios Samaras
2023-07-02

# Social.Media

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

In this report we will be looking at the Social.Media data channel of
the online news popularity data set. This data set looks at a wide range
of variables from 39644 different news articles. The response variable
that we will be focusing on is **shares**. The purpose of this analysis
is to try to predict how many shares a Social.Media article will get
based on the values of those other variables. We will be modeling shares
using two different linear regression models and two ensemble tree based
models.

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

## Social.Media EDA

### Social.Media

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

## Social.Media Summarizations

``` r
#Shares table for data_channel_train
summary_table(data_channel_train)
```

    ##           Shares
    ## Minimum    53.00
    ## Q1       1400.00
    ## Median   2100.00
    ## Q3       3900.00
    ## Maximum 59000.00
    ## Mean     3658.17
    ## SD       5160.17

The above table displays the Social.Media 5-number summary for the
shares. It also includes the mean and standard deviation. Because the
mean is greater than the median, we suspect that the Social.Media shares
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
0.000944
</td>
<td style="text-align:right;">
-0.068884
</td>
<td style="text-align:right;">
-0.040391
</td>
<td style="text-align:right;">
-0.029275
</td>
<td style="text-align:right;">
-0.038731
</td>
<td style="text-align:right;">
-0.018610
</td>
<td style="text-align:right;">
-0.028229
</td>
<td style="text-align:right;">
-0.047191
</td>
<td style="text-align:right;">
0.016953
</td>
<td style="text-align:right;">
-0.093680
</td>
<td style="text-align:right;">
0.012761
</td>
<td style="text-align:right;">
0.018032
</td>
<td style="text-align:right;">
0.013994
</td>
<td style="text-align:right;">
0.096249
</td>
<td style="text-align:right;">
0.094254
</td>
<td style="text-align:right;">
-0.026473
</td>
<td style="text-align:right;">
0.006152
</td>
<td style="text-align:right;">
0.008071
</td>
<td style="text-align:right;">
0.034462
</td>
<td style="text-align:right;">
-0.005113
</td>
<td style="text-align:right;">
0.021461
</td>
<td style="text-align:right;">
-0.045965
</td>
<td style="text-align:right;">
0.071589
</td>
<td style="text-align:right;">
0.041687
</td>
<td style="text-align:right;">
-0.031840
</td>
<td style="text-align:right;">
0.012507
</td>
<td style="text-align:right;">
-0.042673
</td>
<td style="text-align:right;">
-0.019413
</td>
<td style="text-align:right;">
-0.027215
</td>
<td style="text-align:right;">
-0.003254
</td>
<td style="text-align:right;">
-0.014461
</td>
<td style="text-align:right;">
0.004206
</td>
<td style="text-align:right;">
-0.026661
</td>
<td style="text-align:right;">
-0.061533
</td>
<td style="text-align:right;">
0.018167
</td>
<td style="text-align:right;">
-0.009660
</td>
<td style="text-align:right;">
-0.004838
</td>
<td style="text-align:right;">
0.023110
</td>
<td style="text-align:right;">
0.045058
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Content
</td>
<td style="text-align:right;">
0.000944
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.686668
</td>
<td style="text-align:right;">
-0.561367
</td>
<td style="text-align:right;">
0.658262
</td>
<td style="text-align:right;">
0.463212
</td>
<td style="text-align:right;">
0.503601
</td>
<td style="text-align:right;">
0.000255
</td>
<td style="text-align:right;">
0.045661
</td>
<td style="text-align:right;">
0.126748
</td>
<td style="text-align:right;">
-0.001319
</td>
<td style="text-align:right;">
0.003107
</td>
<td style="text-align:right;">
0.000807
</td>
<td style="text-align:right;">
-0.000763
</td>
<td style="text-align:right;">
0.006313
</td>
<td style="text-align:right;">
-0.086825
</td>
<td style="text-align:right;">
-0.055647
</td>
<td style="text-align:right;">
0.004932
</td>
<td style="text-align:right;">
-0.002663
</td>
<td style="text-align:right;">
-0.041068
</td>
<td style="text-align:right;">
0.120019
</td>
<td style="text-align:right;">
0.001814
</td>
<td style="text-align:right;">
-0.074397
</td>
<td style="text-align:right;">
-0.046578
</td>
<td style="text-align:right;">
0.192711
</td>
<td style="text-align:right;">
-0.098840
</td>
<td style="text-align:right;">
0.030998
</td>
<td style="text-align:right;">
0.111615
</td>
<td style="text-align:right;">
0.000062
</td>
<td style="text-align:right;">
0.084415
</td>
<td style="text-align:right;">
0.117882
</td>
<td style="text-align:right;">
-0.038620
</td>
<td style="text-align:right;">
0.086445
</td>
<td style="text-align:right;">
0.102827
</td>
<td style="text-align:right;">
-0.281511
</td>
<td style="text-align:right;">
0.434432
</td>
<td style="text-align:right;">
-0.098763
</td>
<td style="text-align:right;">
-0.488994
</td>
<td style="text-align:right;">
0.246887
</td>
<td style="text-align:right;">
0.035333
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique
</td>
<td style="text-align:right;">
-0.068884
</td>
<td style="text-align:right;">
-0.686668
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.926428
</td>
<td style="text-align:right;">
-0.385933
</td>
<td style="text-align:right;">
-0.286230
</td>
<td style="text-align:right;">
-0.361640
</td>
<td style="text-align:right;">
0.147479
</td>
<td style="text-align:right;">
0.308733
</td>
<td style="text-align:right;">
-0.121633
</td>
<td style="text-align:right;">
-0.023456
</td>
<td style="text-align:right;">
-0.008544
</td>
<td style="text-align:right;">
-0.006565
</td>
<td style="text-align:right;">
-0.010614
</td>
<td style="text-align:right;">
0.019545
</td>
<td style="text-align:right;">
0.097007
</td>
<td style="text-align:right;">
0.088461
</td>
<td style="text-align:right;">
0.003099
</td>
<td style="text-align:right;">
0.031977
</td>
<td style="text-align:right;">
0.005723
</td>
<td style="text-align:right;">
-0.098898
</td>
<td style="text-align:right;">
-0.035303
</td>
<td style="text-align:right;">
-0.021987
</td>
<td style="text-align:right;">
0.046718
</td>
<td style="text-align:right;">
-0.135650
</td>
<td style="text-align:right;">
0.193896
</td>
<td style="text-align:right;">
-0.080025
</td>
<td style="text-align:right;">
0.148311
</td>
<td style="text-align:right;">
0.041039
</td>
<td style="text-align:right;">
0.050837
</td>
<td style="text-align:right;">
0.043769
</td>
<td style="text-align:right;">
0.130234
</td>
<td style="text-align:right;">
0.040809
</td>
<td style="text-align:right;">
0.103214
</td>
<td style="text-align:right;">
0.403185
</td>
<td style="text-align:right;">
-0.268433
</td>
<td style="text-align:right;">
-0.008291
</td>
<td style="text-align:right;">
0.328041
</td>
<td style="text-align:right;">
-0.273257
</td>
<td style="text-align:right;">
0.015635
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique.Nonstop
</td>
<td style="text-align:right;">
-0.040391
</td>
<td style="text-align:right;">
-0.561367
</td>
<td style="text-align:right;">
0.926428
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.357868
</td>
<td style="text-align:right;">
-0.258669
</td>
<td style="text-align:right;">
-0.395207
</td>
<td style="text-align:right;">
0.143072
</td>
<td style="text-align:right;">
0.334716
</td>
<td style="text-align:right;">
-0.126179
</td>
<td style="text-align:right;">
-0.023545
</td>
<td style="text-align:right;">
0.005583
</td>
<td style="text-align:right;">
0.008177
</td>
<td style="text-align:right;">
-0.005550
</td>
<td style="text-align:right;">
0.012121
</td>
<td style="text-align:right;">
0.080109
</td>
<td style="text-align:right;">
0.080401
</td>
<td style="text-align:right;">
0.011641
</td>
<td style="text-align:right;">
0.038985
</td>
<td style="text-align:right;">
0.003128
</td>
<td style="text-align:right;">
-0.089561
</td>
<td style="text-align:right;">
-0.038935
</td>
<td style="text-align:right;">
-0.032413
</td>
<td style="text-align:right;">
0.063056
</td>
<td style="text-align:right;">
-0.127424
</td>
<td style="text-align:right;">
0.173339
</td>
<td style="text-align:right;">
-0.058351
</td>
<td style="text-align:right;">
0.214363
</td>
<td style="text-align:right;">
0.074294
</td>
<td style="text-align:right;">
0.091250
</td>
<td style="text-align:right;">
0.080587
</td>
<td style="text-align:right;">
0.150073
</td>
<td style="text-align:right;">
0.077240
</td>
<td style="text-align:right;">
0.189220
</td>
<td style="text-align:right;">
0.338251
</td>
<td style="text-align:right;">
-0.114314
</td>
<td style="text-align:right;">
-0.054773
</td>
<td style="text-align:right;">
0.202848
</td>
<td style="text-align:right;">
-0.209937
</td>
<td style="text-align:right;">
0.008980
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Links
</td>
<td style="text-align:right;">
-0.029275
</td>
<td style="text-align:right;">
0.658262
</td>
<td style="text-align:right;">
-0.385933
</td>
<td style="text-align:right;">
-0.357868
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.624771
</td>
<td style="text-align:right;">
0.581691
</td>
<td style="text-align:right;">
0.004155
</td>
<td style="text-align:right;">
0.155616
</td>
<td style="text-align:right;">
0.173368
</td>
<td style="text-align:right;">
0.018940
</td>
<td style="text-align:right;">
0.005669
</td>
<td style="text-align:right;">
0.005400
</td>
<td style="text-align:right;">
-0.074102
</td>
<td style="text-align:right;">
0.010244
</td>
<td style="text-align:right;">
-0.135968
</td>
<td style="text-align:right;">
-0.005023
</td>
<td style="text-align:right;">
0.021595
</td>
<td style="text-align:right;">
0.037124
</td>
<td style="text-align:right;">
-0.063789
</td>
<td style="text-align:right;">
0.131085
</td>
<td style="text-align:right;">
-0.019146
</td>
<td style="text-align:right;">
-0.118288
</td>
<td style="text-align:right;">
-0.047639
</td>
<td style="text-align:right;">
0.251007
</td>
<td style="text-align:right;">
-0.015255
</td>
<td style="text-align:right;">
-0.083454
</td>
<td style="text-align:right;">
0.161788
</td>
<td style="text-align:right;">
0.080273
</td>
<td style="text-align:right;">
0.069826
</td>
<td style="text-align:right;">
0.096988
</td>
<td style="text-align:right;">
-0.027709
</td>
<td style="text-align:right;">
0.065573
</td>
<td style="text-align:right;">
0.183652
</td>
<td style="text-align:right;">
-0.175679
</td>
<td style="text-align:right;">
0.343674
</td>
<td style="text-align:right;">
-0.079859
</td>
<td style="text-align:right;">
-0.333871
</td>
<td style="text-align:right;">
0.153577
</td>
<td style="text-align:right;">
0.041272
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Other
</td>
<td style="text-align:right;">
-0.038731
</td>
<td style="text-align:right;">
0.463212
</td>
<td style="text-align:right;">
-0.286230
</td>
<td style="text-align:right;">
-0.258669
</td>
<td style="text-align:right;">
0.624771
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.333818
</td>
<td style="text-align:right;">
0.000607
</td>
<td style="text-align:right;">
0.093367
</td>
<td style="text-align:right;">
0.198637
</td>
<td style="text-align:right;">
0.036191
</td>
<td style="text-align:right;">
-0.004989
</td>
<td style="text-align:right;">
-0.004777
</td>
<td style="text-align:right;">
-0.085198
</td>
<td style="text-align:right;">
-0.016286
</td>
<td style="text-align:right;">
-0.146042
</td>
<td style="text-align:right;">
0.031950
</td>
<td style="text-align:right;">
-0.008028
</td>
<td style="text-align:right;">
-0.002762
</td>
<td style="text-align:right;">
-0.064468
</td>
<td style="text-align:right;">
0.219081
</td>
<td style="text-align:right;">
0.013850
</td>
<td style="text-align:right;">
-0.004228
</td>
<td style="text-align:right;">
-0.088846
</td>
<td style="text-align:right;">
0.104776
</td>
<td style="text-align:right;">
-0.113095
</td>
<td style="text-align:right;">
0.080542
</td>
<td style="text-align:right;">
0.055771
</td>
<td style="text-align:right;">
0.070478
</td>
<td style="text-align:right;">
0.133566
</td>
<td style="text-align:right;">
0.016976
</td>
<td style="text-align:right;">
0.063227
</td>
<td style="text-align:right;">
-0.036566
</td>
<td style="text-align:right;">
0.047710
</td>
<td style="text-align:right;">
-0.152059
</td>
<td style="text-align:right;">
0.228867
</td>
<td style="text-align:right;">
-0.017300
</td>
<td style="text-align:right;">
-0.189858
</td>
<td style="text-align:right;">
0.114013
</td>
<td style="text-align:right;">
-0.025830
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Images
</td>
<td style="text-align:right;">
-0.018610
</td>
<td style="text-align:right;">
0.503601
</td>
<td style="text-align:right;">
-0.361640
</td>
<td style="text-align:right;">
-0.395207
</td>
<td style="text-align:right;">
0.581691
</td>
<td style="text-align:right;">
0.333818
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.104251
</td>
<td style="text-align:right;">
0.043841
</td>
<td style="text-align:right;">
0.063113
</td>
<td style="text-align:right;">
0.020064
</td>
<td style="text-align:right;">
-0.014597
</td>
<td style="text-align:right;">
-0.006154
</td>
<td style="text-align:right;">
-0.026536
</td>
<td style="text-align:right;">
0.005806
</td>
<td style="text-align:right;">
-0.110103
</td>
<td style="text-align:right;">
0.007535
</td>
<td style="text-align:right;">
0.016888
</td>
<td style="text-align:right;">
0.031949
</td>
<td style="text-align:right;">
-0.015292
</td>
<td style="text-align:right;">
0.095174
</td>
<td style="text-align:right;">
0.017005
</td>
<td style="text-align:right;">
-0.182370
</td>
<td style="text-align:right;">
-0.015313
</td>
<td style="text-align:right;">
0.258287
</td>
<td style="text-align:right;">
0.011806
</td>
<td style="text-align:right;">
-0.051641
</td>
<td style="text-align:right;">
0.131823
</td>
<td style="text-align:right;">
0.067030
</td>
<td style="text-align:right;">
0.051219
</td>
<td style="text-align:right;">
0.071798
</td>
<td style="text-align:right;">
-0.021570
</td>
<td style="text-align:right;">
0.039967
</td>
<td style="text-align:right;">
0.141501
</td>
<td style="text-align:right;">
-0.059605
</td>
<td style="text-align:right;">
0.212602
</td>
<td style="text-align:right;">
-0.024237
</td>
<td style="text-align:right;">
-0.189987
</td>
<td style="text-align:right;">
0.093048
</td>
<td style="text-align:right;">
0.084759
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Videos
</td>
<td style="text-align:right;">
-0.028229
</td>
<td style="text-align:right;">
0.000255
</td>
<td style="text-align:right;">
0.147479
</td>
<td style="text-align:right;">
0.143072
</td>
<td style="text-align:right;">
0.004155
</td>
<td style="text-align:right;">
0.000607
</td>
<td style="text-align:right;">
-0.104251
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.026331
</td>
<td style="text-align:right;">
0.066105
</td>
<td style="text-align:right;">
-0.050460
</td>
<td style="text-align:right;">
0.008342
</td>
<td style="text-align:right;">
-0.004765
</td>
<td style="text-align:right;">
-0.035094
</td>
<td style="text-align:right;">
0.039970
</td>
<td style="text-align:right;">
0.025609
</td>
<td style="text-align:right;">
0.043194
</td>
<td style="text-align:right;">
0.030093
</td>
<td style="text-align:right;">
0.052585
</td>
<td style="text-align:right;">
-0.017606
</td>
<td style="text-align:right;">
0.014931
</td>
<td style="text-align:right;">
-0.014957
</td>
<td style="text-align:right;">
-0.106209
</td>
<td style="text-align:right;">
0.023136
</td>
<td style="text-align:right;">
-0.009626
</td>
<td style="text-align:right;">
0.162094
</td>
<td style="text-align:right;">
-0.053046
</td>
<td style="text-align:right;">
0.143973
</td>
<td style="text-align:right;">
0.103311
</td>
<td style="text-align:right;">
0.095525
</td>
<td style="text-align:right;">
0.011801
</td>
<td style="text-align:right;">
0.027962
</td>
<td style="text-align:right;">
-0.020238
</td>
<td style="text-align:right;">
0.128134
</td>
<td style="text-align:right;">
0.101776
</td>
<td style="text-align:right;">
0.097821
</td>
<td style="text-align:right;">
-0.048857
</td>
<td style="text-align:right;">
-0.012029
</td>
<td style="text-align:right;">
-0.028249
</td>
<td style="text-align:right;">
0.058564
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Words
</td>
<td style="text-align:right;">
-0.047191
</td>
<td style="text-align:right;">
0.045661
</td>
<td style="text-align:right;">
0.308733
</td>
<td style="text-align:right;">
0.334716
</td>
<td style="text-align:right;">
0.155616
</td>
<td style="text-align:right;">
0.093367
</td>
<td style="text-align:right;">
0.043841
</td>
<td style="text-align:right;">
-0.026331
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.047511
</td>
<td style="text-align:right;">
-0.041647
</td>
<td style="text-align:right;">
0.001538
</td>
<td style="text-align:right;">
0.000165
</td>
<td style="text-align:right;">
-0.027317
</td>
<td style="text-align:right;">
0.035826
</td>
<td style="text-align:right;">
-0.010518
</td>
<td style="text-align:right;">
-0.005900
</td>
<td style="text-align:right;">
-0.012599
</td>
<td style="text-align:right;">
-0.036676
</td>
<td style="text-align:right;">
-0.004194
</td>
<td style="text-align:right;">
0.022538
</td>
<td style="text-align:right;">
0.006182
</td>
<td style="text-align:right;">
0.054138
</td>
<td style="text-align:right;">
-0.065899
</td>
<td style="text-align:right;">
0.048133
</td>
<td style="text-align:right;">
-0.064123
</td>
<td style="text-align:right;">
-0.015882
</td>
<td style="text-align:right;">
0.230634
</td>
<td style="text-align:right;">
0.082968
</td>
<td style="text-align:right;">
0.161050
</td>
<td style="text-align:right;">
0.070748
</td>
<td style="text-align:right;">
0.317581
</td>
<td style="text-align:right;">
0.107287
</td>
<td style="text-align:right;">
0.210778
</td>
<td style="text-align:right;">
0.045428
</td>
<td style="text-align:right;">
0.201443
</td>
<td style="text-align:right;">
-0.098138
</td>
<td style="text-align:right;">
-0.083225
</td>
<td style="text-align:right;">
-0.059452
</td>
<td style="text-align:right;">
-0.003999
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Key
</td>
<td style="text-align:right;">
0.016953
</td>
<td style="text-align:right;">
0.126748
</td>
<td style="text-align:right;">
-0.121633
</td>
<td style="text-align:right;">
-0.126179
</td>
<td style="text-align:right;">
0.173368
</td>
<td style="text-align:right;">
0.198637
</td>
<td style="text-align:right;">
0.063113
</td>
<td style="text-align:right;">
0.066105
</td>
<td style="text-align:right;">
0.047511
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.002020
</td>
<td style="text-align:right;">
0.031950
</td>
<td style="text-align:right;">
0.004514
</td>
<td style="text-align:right;">
-0.434761
</td>
<td style="text-align:right;">
0.022165
</td>
<td style="text-align:right;">
-0.410802
</td>
<td style="text-align:right;">
-0.369723
</td>
<td style="text-align:right;">
0.081040
</td>
<td style="text-align:right;">
-0.055346
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
0.055952
</td>
<td style="text-align:right;">
0.023691
</td>
<td style="text-align:right;">
-0.042312
</td>
<td style="text-align:right;">
-0.149375
</td>
<td style="text-align:right;">
0.022368
</td>
<td style="text-align:right;">
0.042726
</td>
<td style="text-align:right;">
0.077732
</td>
<td style="text-align:right;">
0.023737
</td>
<td style="text-align:right;">
0.102337
</td>
<td style="text-align:right;">
0.082372
</td>
<td style="text-align:right;">
-0.072845
</td>
<td style="text-align:right;">
0.088671
</td>
<td style="text-align:right;">
-0.092216
</td>
<td style="text-align:right;">
0.050772
</td>
<td style="text-align:right;">
-0.138082
</td>
<td style="text-align:right;">
0.113028
</td>
<td style="text-align:right;">
-0.037944
</td>
<td style="text-align:right;">
-0.084582
</td>
<td style="text-align:right;">
0.035014
</td>
<td style="text-align:right;">
-0.032221
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Worst.Key
</td>
<td style="text-align:right;">
-0.093680
</td>
<td style="text-align:right;">
-0.001319
</td>
<td style="text-align:right;">
-0.023456
</td>
<td style="text-align:right;">
-0.023545
</td>
<td style="text-align:right;">
0.018940
</td>
<td style="text-align:right;">
0.036191
</td>
<td style="text-align:right;">
0.020064
</td>
<td style="text-align:right;">
-0.050460
</td>
<td style="text-align:right;">
-0.041647
</td>
<td style="text-align:right;">
0.002020
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.002876
</td>
<td style="text-align:right;">
0.044756
</td>
<td style="text-align:right;">
-0.074470
</td>
<td style="text-align:right;">
-0.855507
</td>
<td style="text-align:right;">
-0.487217
</td>
<td style="text-align:right;">
-0.143723
</td>
<td style="text-align:right;">
-0.060297
</td>
<td style="text-align:right;">
-0.182303
</td>
<td style="text-align:right;">
-0.032592
</td>
<td style="text-align:right;">
-0.037628
</td>
<td style="text-align:right;">
-0.045579
</td>
<td style="text-align:right;">
0.059724
</td>
<td style="text-align:right;">
0.075924
</td>
<td style="text-align:right;">
-0.107324
</td>
<td style="text-align:right;">
-0.019037
</td>
<td style="text-align:right;">
0.018961
</td>
<td style="text-align:right;">
-0.054772
</td>
<td style="text-align:right;">
0.020125
</td>
<td style="text-align:right;">
0.003283
</td>
<td style="text-align:right;">
-0.035970
</td>
<td style="text-align:right;">
0.024376
</td>
<td style="text-align:right;">
-0.044075
</td>
<td style="text-align:right;">
-0.032639
</td>
<td style="text-align:right;">
-0.037726
</td>
<td style="text-align:right;">
-0.024010
</td>
<td style="text-align:right;">
0.031647
</td>
<td style="text-align:right;">
0.019236
</td>
<td style="text-align:right;">
0.006246
</td>
<td style="text-align:right;">
0.003693
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Worst.Key
</td>
<td style="text-align:right;">
0.012761
</td>
<td style="text-align:right;">
0.003107
</td>
<td style="text-align:right;">
-0.008544
</td>
<td style="text-align:right;">
0.005583
</td>
<td style="text-align:right;">
0.005669
</td>
<td style="text-align:right;">
-0.004989
</td>
<td style="text-align:right;">
-0.014597
</td>
<td style="text-align:right;">
0.008342
</td>
<td style="text-align:right;">
0.001538
</td>
<td style="text-align:right;">
0.031950
</td>
<td style="text-align:right;">
-0.002876
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.978774
</td>
<td style="text-align:right;">
-0.050759
</td>
<td style="text-align:right;">
0.011440
</td>
<td style="text-align:right;">
-0.056654
</td>
<td style="text-align:right;">
-0.047594
</td>
<td style="text-align:right;">
0.804291
</td>
<td style="text-align:right;">
0.703510
</td>
<td style="text-align:right;">
-0.001114
</td>
<td style="text-align:right;">
-0.003252
</td>
<td style="text-align:right;">
-0.004259
</td>
<td style="text-align:right;">
0.015344
</td>
<td style="text-align:right;">
-0.016855
</td>
<td style="text-align:right;">
-0.011794
</td>
<td style="text-align:right;">
0.015102
</td>
<td style="text-align:right;">
-0.017233
</td>
<td style="text-align:right;">
-0.015499
</td>
<td style="text-align:right;">
0.026132
</td>
<td style="text-align:right;">
0.007104
</td>
<td style="text-align:right;">
-0.002184
</td>
<td style="text-align:right;">
0.012645
</td>
<td style="text-align:right;">
-0.013868
</td>
<td style="text-align:right;">
-0.009489
</td>
<td style="text-align:right;">
-0.026971
</td>
<td style="text-align:right;">
0.039833
</td>
<td style="text-align:right;">
0.019775
</td>
<td style="text-align:right;">
0.010582
</td>
<td style="text-align:right;">
0.006722
</td>
<td style="text-align:right;">
-0.010504
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Worst.Key
</td>
<td style="text-align:right;">
0.018032
</td>
<td style="text-align:right;">
0.000807
</td>
<td style="text-align:right;">
-0.006565
</td>
<td style="text-align:right;">
0.008177
</td>
<td style="text-align:right;">
0.005400
</td>
<td style="text-align:right;">
-0.004777
</td>
<td style="text-align:right;">
-0.006154
</td>
<td style="text-align:right;">
-0.004765
</td>
<td style="text-align:right;">
0.000165
</td>
<td style="text-align:right;">
0.004514
</td>
<td style="text-align:right;">
0.044756
</td>
<td style="text-align:right;">
0.978774
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.059948
</td>
<td style="text-align:right;">
-0.041792
</td>
<td style="text-align:right;">
-0.091281
</td>
<td style="text-align:right;">
-0.061731
</td>
<td style="text-align:right;">
0.791916
</td>
<td style="text-align:right;">
0.679419
</td>
<td style="text-align:right;">
0.004249
</td>
<td style="text-align:right;">
0.001203
</td>
<td style="text-align:right;">
0.000378
</td>
<td style="text-align:right;">
0.019776
</td>
<td style="text-align:right;">
-0.008772
</td>
<td style="text-align:right;">
-0.012723
</td>
<td style="text-align:right;">
0.009839
</td>
<td style="text-align:right;">
-0.021197
</td>
<td style="text-align:right;">
-0.021655
</td>
<td style="text-align:right;">
0.024111
</td>
<td style="text-align:right;">
0.007055
</td>
<td style="text-align:right;">
0.002663
</td>
<td style="text-align:right;">
0.009106
</td>
<td style="text-align:right;">
-0.010119
</td>
<td style="text-align:right;">
-0.012871
</td>
<td style="text-align:right;">
-0.028426
</td>
<td style="text-align:right;">
0.041878
</td>
<td style="text-align:right;">
0.026844
</td>
<td style="text-align:right;">
0.019742
</td>
<td style="text-align:right;">
0.005138
</td>
<td style="text-align:right;">
-0.010022
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Best.Key
</td>
<td style="text-align:right;">
0.013994
</td>
<td style="text-align:right;">
-0.000763
</td>
<td style="text-align:right;">
-0.010614
</td>
<td style="text-align:right;">
-0.005550
</td>
<td style="text-align:right;">
-0.074102
</td>
<td style="text-align:right;">
-0.085198
</td>
<td style="text-align:right;">
-0.026536
</td>
<td style="text-align:right;">
-0.035094
</td>
<td style="text-align:right;">
-0.027317
</td>
<td style="text-align:right;">
-0.434761
</td>
<td style="text-align:right;">
-0.074470
</td>
<td style="text-align:right;">
-0.050759
</td>
<td style="text-align:right;">
-0.059948
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.078761
</td>
<td style="text-align:right;">
0.665896
</td>
<td style="text-align:right;">
0.354051
</td>
<td style="text-align:right;">
-0.052956
</td>
<td style="text-align:right;">
0.071083
</td>
<td style="text-align:right;">
-0.024512
</td>
<td style="text-align:right;">
-0.036123
</td>
<td style="text-align:right;">
-0.031435
</td>
<td style="text-align:right;">
-0.080172
</td>
<td style="text-align:right;">
0.148536
</td>
<td style="text-align:right;">
-0.025947
</td>
<td style="text-align:right;">
0.018556
</td>
<td style="text-align:right;">
0.033226
</td>
<td style="text-align:right;">
-0.057217
</td>
<td style="text-align:right;">
-0.049430
</td>
<td style="text-align:right;">
-0.040781
</td>
<td style="text-align:right;">
0.019348
</td>
<td style="text-align:right;">
-0.036565
</td>
<td style="text-align:right;">
0.016700
</td>
<td style="text-align:right;">
-0.029721
</td>
<td style="text-align:right;">
0.031287
</td>
<td style="text-align:right;">
-0.026794
</td>
<td style="text-align:right;">
0.008521
</td>
<td style="text-align:right;">
-0.015968
</td>
<td style="text-align:right;">
0.033554
</td>
<td style="text-align:right;">
0.021113
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Best.Key
</td>
<td style="text-align:right;">
0.096249
</td>
<td style="text-align:right;">
0.006313
</td>
<td style="text-align:right;">
0.019545
</td>
<td style="text-align:right;">
0.012121
</td>
<td style="text-align:right;">
0.010244
</td>
<td style="text-align:right;">
-0.016286
</td>
<td style="text-align:right;">
0.005806
</td>
<td style="text-align:right;">
0.039970
</td>
<td style="text-align:right;">
0.035826
</td>
<td style="text-align:right;">
0.022165
</td>
<td style="text-align:right;">
-0.855507
</td>
<td style="text-align:right;">
0.011440
</td>
<td style="text-align:right;">
-0.041792
</td>
<td style="text-align:right;">
0.078761
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.539139
</td>
<td style="text-align:right;">
0.148440
</td>
<td style="text-align:right;">
0.079277
</td>
<td style="text-align:right;">
0.213074
</td>
<td style="text-align:right;">
0.034441
</td>
<td style="text-align:right;">
0.045779
</td>
<td style="text-align:right;">
0.054596
</td>
<td style="text-align:right;">
-0.045142
</td>
<td style="text-align:right;">
-0.028210
</td>
<td style="text-align:right;">
0.107266
</td>
<td style="text-align:right;">
-0.030290
</td>
<td style="text-align:right;">
-0.007801
</td>
<td style="text-align:right;">
0.054425
</td>
<td style="text-align:right;">
-0.009011
</td>
<td style="text-align:right;">
0.009949
</td>
<td style="text-align:right;">
0.014265
</td>
<td style="text-align:right;">
0.003478
</td>
<td style="text-align:right;">
0.011007
</td>
<td style="text-align:right;">
0.019560
</td>
<td style="text-align:right;">
0.019266
</td>
<td style="text-align:right;">
0.021324
</td>
<td style="text-align:right;">
-0.033717
</td>
<td style="text-align:right;">
-0.014007
</td>
<td style="text-align:right;">
-0.017239
</td>
<td style="text-align:right;">
-0.003558
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Best.Key
</td>
<td style="text-align:right;">
0.094254
</td>
<td style="text-align:right;">
-0.086825
</td>
<td style="text-align:right;">
0.097007
</td>
<td style="text-align:right;">
0.080109
</td>
<td style="text-align:right;">
-0.135968
</td>
<td style="text-align:right;">
-0.146042
</td>
<td style="text-align:right;">
-0.110103
</td>
<td style="text-align:right;">
0.025609
</td>
<td style="text-align:right;">
-0.010518
</td>
<td style="text-align:right;">
-0.410802
</td>
<td style="text-align:right;">
-0.487217
</td>
<td style="text-align:right;">
-0.056654
</td>
<td style="text-align:right;">
-0.091281
</td>
<td style="text-align:right;">
0.665896
</td>
<td style="text-align:right;">
0.539139
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.452166
</td>
<td style="text-align:right;">
0.014307
</td>
<td style="text-align:right;">
0.254006
</td>
<td style="text-align:right;">
0.023360
</td>
<td style="text-align:right;">
0.013239
</td>
<td style="text-align:right;">
0.033303
</td>
<td style="text-align:right;">
-0.075315
</td>
<td style="text-align:right;">
0.070920
</td>
<td style="text-align:right;">
-0.051399
</td>
<td style="text-align:right;">
0.119711
</td>
<td style="text-align:right;">
-0.023578
</td>
<td style="text-align:right;">
0.000478
</td>
<td style="text-align:right;">
-0.092663
</td>
<td style="text-align:right;">
-0.045841
</td>
<td style="text-align:right;">
0.062457
</td>
<td style="text-align:right;">
-0.065120
</td>
<td style="text-align:right;">
0.057692
</td>
<td style="text-align:right;">
-0.033693
</td>
<td style="text-align:right;">
0.060646
</td>
<td style="text-align:right;">
-0.062983
</td>
<td style="text-align:right;">
-0.037725
</td>
<td style="text-align:right;">
-0.003313
</td>
<td style="text-align:right;">
-0.018002
</td>
<td style="text-align:right;">
0.016522
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Min.Key
</td>
<td style="text-align:right;">
-0.026473
</td>
<td style="text-align:right;">
-0.055647
</td>
<td style="text-align:right;">
0.088461
</td>
<td style="text-align:right;">
0.080401
</td>
<td style="text-align:right;">
-0.005023
</td>
<td style="text-align:right;">
0.031950
</td>
<td style="text-align:right;">
0.007535
</td>
<td style="text-align:right;">
0.043194
</td>
<td style="text-align:right;">
-0.005900
</td>
<td style="text-align:right;">
-0.369723
</td>
<td style="text-align:right;">
-0.143723
</td>
<td style="text-align:right;">
-0.047594
</td>
<td style="text-align:right;">
-0.061731
</td>
<td style="text-align:right;">
0.354051
</td>
<td style="text-align:right;">
0.148440
</td>
<td style="text-align:right;">
0.452166
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.001603
</td>
<td style="text-align:right;">
0.383869
</td>
<td style="text-align:right;">
0.001734
</td>
<td style="text-align:right;">
0.024343
</td>
<td style="text-align:right;">
0.022650
</td>
<td style="text-align:right;">
0.090389
</td>
<td style="text-align:right;">
0.022661
</td>
<td style="text-align:right;">
-0.174504
</td>
<td style="text-align:right;">
0.022436
</td>
<td style="text-align:right;">
0.037051
</td>
<td style="text-align:right;">
0.036209
</td>
<td style="text-align:right;">
-0.057650
</td>
<td style="text-align:right;">
-0.027393
</td>
<td style="text-align:right;">
0.060514
</td>
<td style="text-align:right;">
-0.047168
</td>
<td style="text-align:right;">
0.066370
</td>
<td style="text-align:right;">
0.013669
</td>
<td style="text-align:right;">
0.065086
</td>
<td style="text-align:right;">
-0.017274
</td>
<td style="text-align:right;">
-0.038474
</td>
<td style="text-align:right;">
-0.018230
</td>
<td style="text-align:right;">
-0.039292
</td>
<td style="text-align:right;">
0.042713
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Max.Key
</td>
<td style="text-align:right;">
0.006152
</td>
<td style="text-align:right;">
0.004932
</td>
<td style="text-align:right;">
0.003099
</td>
<td style="text-align:right;">
0.011641
</td>
<td style="text-align:right;">
0.021595
</td>
<td style="text-align:right;">
-0.008028
</td>
<td style="text-align:right;">
0.016888
</td>
<td style="text-align:right;">
0.030093
</td>
<td style="text-align:right;">
-0.012599
</td>
<td style="text-align:right;">
0.081040
</td>
<td style="text-align:right;">
-0.060297
</td>
<td style="text-align:right;">
0.804291
</td>
<td style="text-align:right;">
0.791916
</td>
<td style="text-align:right;">
-0.052956
</td>
<td style="text-align:right;">
0.079277
</td>
<td style="text-align:right;">
0.014307
</td>
<td style="text-align:right;">
-0.001603
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.853927
</td>
<td style="text-align:right;">
0.000996
</td>
<td style="text-align:right;">
0.156403
</td>
<td style="text-align:right;">
0.145022
</td>
<td style="text-align:right;">
-0.021613
</td>
<td style="text-align:right;">
-0.034475
</td>
<td style="text-align:right;">
-0.038100
</td>
<td style="text-align:right;">
0.128448
</td>
<td style="text-align:right;">
-0.064459
</td>
<td style="text-align:right;">
-0.001842
</td>
<td style="text-align:right;">
0.017160
</td>
<td style="text-align:right;">
-0.015301
</td>
<td style="text-align:right;">
0.002668
</td>
<td style="text-align:right;">
-0.016730
</td>
<td style="text-align:right;">
0.007991
</td>
<td style="text-align:right;">
0.011598
</td>
<td style="text-align:right;">
-0.008984
</td>
<td style="text-align:right;">
0.050004
</td>
<td style="text-align:right;">
-0.005933
</td>
<td style="text-align:right;">
0.000746
</td>
<td style="text-align:right;">
-0.006929
</td>
<td style="text-align:right;">
0.004465
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Avg.Key
</td>
<td style="text-align:right;">
0.008071
</td>
<td style="text-align:right;">
-0.002663
</td>
<td style="text-align:right;">
0.031977
</td>
<td style="text-align:right;">
0.038985
</td>
<td style="text-align:right;">
0.037124
</td>
<td style="text-align:right;">
-0.002762
</td>
<td style="text-align:right;">
0.031949
</td>
<td style="text-align:right;">
0.052585
</td>
<td style="text-align:right;">
-0.036676
</td>
<td style="text-align:right;">
-0.055346
</td>
<td style="text-align:right;">
-0.182303
</td>
<td style="text-align:right;">
0.703510
</td>
<td style="text-align:right;">
0.679419
</td>
<td style="text-align:right;">
0.071083
</td>
<td style="text-align:right;">
0.213074
</td>
<td style="text-align:right;">
0.254006
</td>
<td style="text-align:right;">
0.383869
</td>
<td style="text-align:right;">
0.853927
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.010912
</td>
<td style="text-align:right;">
0.115195
</td>
<td style="text-align:right;">
0.105426
</td>
<td style="text-align:right;">
0.015122
</td>
<td style="text-align:right;">
-0.062320
</td>
<td style="text-align:right;">
-0.126514
</td>
<td style="text-align:right;">
0.189739
</td>
<td style="text-align:right;">
-0.071381
</td>
<td style="text-align:right;">
0.043780
</td>
<td style="text-align:right;">
0.007314
</td>
<td style="text-align:right;">
0.010108
</td>
<td style="text-align:right;">
0.043646
</td>
<td style="text-align:right;">
-0.031445
</td>
<td style="text-align:right;">
0.024334
</td>
<td style="text-align:right;">
0.032694
</td>
<td style="text-align:right;">
0.013512
</td>
<td style="text-align:right;">
0.047350
</td>
<td style="text-align:right;">
-0.027477
</td>
<td style="text-align:right;">
-0.022950
</td>
<td style="text-align:right;">
-0.008937
</td>
<td style="text-align:right;">
0.040571
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Ref
</td>
<td style="text-align:right;">
0.034462
</td>
<td style="text-align:right;">
-0.041068
</td>
<td style="text-align:right;">
0.005723
</td>
<td style="text-align:right;">
0.003128
</td>
<td style="text-align:right;">
-0.063789
</td>
<td style="text-align:right;">
-0.064468
</td>
<td style="text-align:right;">
-0.015292
</td>
<td style="text-align:right;">
-0.017606
</td>
<td style="text-align:right;">
-0.004194
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
-0.032592
</td>
<td style="text-align:right;">
-0.001114
</td>
<td style="text-align:right;">
0.004249
</td>
<td style="text-align:right;">
-0.024512
</td>
<td style="text-align:right;">
0.034441
</td>
<td style="text-align:right;">
0.023360
</td>
<td style="text-align:right;">
0.001734
</td>
<td style="text-align:right;">
0.000996
</td>
<td style="text-align:right;">
0.010912
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.420597
</td>
<td style="text-align:right;">
0.820020
</td>
<td style="text-align:right;">
0.067729
</td>
<td style="text-align:right;">
0.071118
</td>
<td style="text-align:right;">
-0.068390
</td>
<td style="text-align:right;">
-0.026244
</td>
<td style="text-align:right;">
-0.029323
</td>
<td style="text-align:right;">
0.018144
</td>
<td style="text-align:right;">
-0.010294
</td>
<td style="text-align:right;">
0.008405
</td>
<td style="text-align:right;">
0.020313
</td>
<td style="text-align:right;">
0.003641
</td>
<td style="text-align:right;">
0.006165
</td>
<td style="text-align:right;">
-0.008927
</td>
<td style="text-align:right;">
-0.029373
</td>
<td style="text-align:right;">
0.019369
</td>
<td style="text-align:right;">
-0.045621
</td>
<td style="text-align:right;">
-0.026125
</td>
<td style="text-align:right;">
-0.015282
</td>
<td style="text-align:right;">
-0.006702
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Ref
</td>
<td style="text-align:right;">
-0.005113
</td>
<td style="text-align:right;">
0.120019
</td>
<td style="text-align:right;">
-0.098898
</td>
<td style="text-align:right;">
-0.089561
</td>
<td style="text-align:right;">
0.131085
</td>
<td style="text-align:right;">
0.219081
</td>
<td style="text-align:right;">
0.095174
</td>
<td style="text-align:right;">
0.014931
</td>
<td style="text-align:right;">
0.022538
</td>
<td style="text-align:right;">
0.055952
</td>
<td style="text-align:right;">
-0.037628
</td>
<td style="text-align:right;">
-0.003252
</td>
<td style="text-align:right;">
0.001203
</td>
<td style="text-align:right;">
-0.036123
</td>
<td style="text-align:right;">
0.045779
</td>
<td style="text-align:right;">
0.013239
</td>
<td style="text-align:right;">
0.024343
</td>
<td style="text-align:right;">
0.156403
</td>
<td style="text-align:right;">
0.115195
</td>
<td style="text-align:right;">
0.420597
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.780070
</td>
<td style="text-align:right;">
0.041714
</td>
<td style="text-align:right;">
-0.028871
</td>
<td style="text-align:right;">
0.030864
</td>
<td style="text-align:right;">
-0.068364
</td>
<td style="text-align:right;">
0.005978
</td>
<td style="text-align:right;">
0.005552
</td>
<td style="text-align:right;">
0.017693
</td>
<td style="text-align:right;">
0.030309
</td>
<td style="text-align:right;">
0.001111
</td>
<td style="text-align:right;">
0.037679
</td>
<td style="text-align:right;">
-0.026934
</td>
<td style="text-align:right;">
0.008033
</td>
<td style="text-align:right;">
-0.071943
</td>
<td style="text-align:right;">
0.073637
</td>
<td style="text-align:right;">
-0.028245
</td>
<td style="text-align:right;">
-0.088865
</td>
<td style="text-align:right;">
0.026667
</td>
<td style="text-align:right;">
0.010934
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Ref
</td>
<td style="text-align:right;">
0.021461
</td>
<td style="text-align:right;">
0.001814
</td>
<td style="text-align:right;">
-0.035303
</td>
<td style="text-align:right;">
-0.038935
</td>
<td style="text-align:right;">
-0.019146
</td>
<td style="text-align:right;">
0.013850
</td>
<td style="text-align:right;">
0.017005
</td>
<td style="text-align:right;">
-0.014957
</td>
<td style="text-align:right;">
0.006182
</td>
<td style="text-align:right;">
0.023691
</td>
<td style="text-align:right;">
-0.045579
</td>
<td style="text-align:right;">
-0.004259
</td>
<td style="text-align:right;">
0.000378
</td>
<td style="text-align:right;">
-0.031435
</td>
<td style="text-align:right;">
0.054596
</td>
<td style="text-align:right;">
0.033303
</td>
<td style="text-align:right;">
0.022650
</td>
<td style="text-align:right;">
0.145022
</td>
<td style="text-align:right;">
0.105426
</td>
<td style="text-align:right;">
0.820020
</td>
<td style="text-align:right;">
0.780070
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.069219
</td>
<td style="text-align:right;">
0.025727
</td>
<td style="text-align:right;">
-0.026487
</td>
<td style="text-align:right;">
-0.049053
</td>
<td style="text-align:right;">
-0.024722
</td>
<td style="text-align:right;">
0.004360
</td>
<td style="text-align:right;">
-0.002259
</td>
<td style="text-align:right;">
0.003404
</td>
<td style="text-align:right;">
0.004595
</td>
<td style="text-align:right;">
0.025854
</td>
<td style="text-align:right;">
-0.013936
</td>
<td style="text-align:right;">
-0.011644
</td>
<td style="text-align:right;">
-0.050623
</td>
<td style="text-align:right;">
0.034292
</td>
<td style="text-align:right;">
-0.036237
</td>
<td style="text-align:right;">
-0.039568
</td>
<td style="text-align:right;">
-0.005448
</td>
<td style="text-align:right;">
-0.004049
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_00
</td>
<td style="text-align:right;">
-0.045965
</td>
<td style="text-align:right;">
-0.074397
</td>
<td style="text-align:right;">
-0.021987
</td>
<td style="text-align:right;">
-0.032413
</td>
<td style="text-align:right;">
-0.118288
</td>
<td style="text-align:right;">
-0.004228
</td>
<td style="text-align:right;">
-0.182370
</td>
<td style="text-align:right;">
-0.106209
</td>
<td style="text-align:right;">
0.054138
</td>
<td style="text-align:right;">
-0.042312
</td>
<td style="text-align:right;">
0.059724
</td>
<td style="text-align:right;">
0.015344
</td>
<td style="text-align:right;">
0.019776
</td>
<td style="text-align:right;">
-0.080172
</td>
<td style="text-align:right;">
-0.045142
</td>
<td style="text-align:right;">
-0.075315
</td>
<td style="text-align:right;">
0.090389
</td>
<td style="text-align:right;">
-0.021613
</td>
<td style="text-align:right;">
0.015122
</td>
<td style="text-align:right;">
0.067729
</td>
<td style="text-align:right;">
0.041714
</td>
<td style="text-align:right;">
0.069219
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.174751
</td>
<td style="text-align:right;">
-0.413846
</td>
<td style="text-align:right;">
-0.495824
</td>
<td style="text-align:right;">
-0.239921
</td>
<td style="text-align:right;">
-0.116692
</td>
<td style="text-align:right;">
-0.082915
</td>
<td style="text-align:right;">
0.018877
</td>
<td style="text-align:right;">
0.007736
</td>
<td style="text-align:right;">
0.016284
</td>
<td style="text-align:right;">
-0.001099
</td>
<td style="text-align:right;">
-0.128565
</td>
<td style="text-align:right;">
-0.102799
</td>
<td style="text-align:right;">
-0.044806
</td>
<td style="text-align:right;">
-0.018766
</td>
<td style="text-align:right;">
-0.001788
</td>
<td style="text-align:right;">
-0.047151
</td>
<td style="text-align:right;">
-0.102330
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_01
</td>
<td style="text-align:right;">
0.071589
</td>
<td style="text-align:right;">
-0.046578
</td>
<td style="text-align:right;">
0.046718
</td>
<td style="text-align:right;">
0.063056
</td>
<td style="text-align:right;">
-0.047639
</td>
<td style="text-align:right;">
-0.088846
</td>
<td style="text-align:right;">
-0.015313
</td>
<td style="text-align:right;">
0.023136
</td>
<td style="text-align:right;">
-0.065899
</td>
<td style="text-align:right;">
-0.149375
</td>
<td style="text-align:right;">
0.075924
</td>
<td style="text-align:right;">
-0.016855
</td>
<td style="text-align:right;">
-0.008772
</td>
<td style="text-align:right;">
0.148536
</td>
<td style="text-align:right;">
-0.028210
</td>
<td style="text-align:right;">
0.070920
</td>
<td style="text-align:right;">
0.022661
</td>
<td style="text-align:right;">
-0.034475
</td>
<td style="text-align:right;">
-0.062320
</td>
<td style="text-align:right;">
0.071118
</td>
<td style="text-align:right;">
-0.028871
</td>
<td style="text-align:right;">
0.025727
</td>
<td style="text-align:right;">
-0.174751
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.140958
</td>
<td style="text-align:right;">
-0.056717
</td>
<td style="text-align:right;">
-0.135426
</td>
<td style="text-align:right;">
0.015470
</td>
<td style="text-align:right;">
0.029695
</td>
<td style="text-align:right;">
-0.002271
</td>
<td style="text-align:right;">
0.006481
</td>
<td style="text-align:right;">
-0.001291
</td>
<td style="text-align:right;">
-0.014781
</td>
<td style="text-align:right;">
0.023869
</td>
<td style="text-align:right;">
0.010069
</td>
<td style="text-align:right;">
0.014586
</td>
<td style="text-align:right;">
-0.027742
</td>
<td style="text-align:right;">
-0.001795
</td>
<td style="text-align:right;">
-0.034388
</td>
<td style="text-align:right;">
0.035696
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_02
</td>
<td style="text-align:right;">
0.041687
</td>
<td style="text-align:right;">
0.192711
</td>
<td style="text-align:right;">
-0.135650
</td>
<td style="text-align:right;">
-0.127424
</td>
<td style="text-align:right;">
0.251007
</td>
<td style="text-align:right;">
0.104776
</td>
<td style="text-align:right;">
0.258287
</td>
<td style="text-align:right;">
-0.009626
</td>
<td style="text-align:right;">
0.048133
</td>
<td style="text-align:right;">
0.022368
</td>
<td style="text-align:right;">
-0.107324
</td>
<td style="text-align:right;">
-0.011794
</td>
<td style="text-align:right;">
-0.012723
</td>
<td style="text-align:right;">
-0.025947
</td>
<td style="text-align:right;">
0.107266
</td>
<td style="text-align:right;">
-0.051399
</td>
<td style="text-align:right;">
-0.174504
</td>
<td style="text-align:right;">
-0.038100
</td>
<td style="text-align:right;">
-0.126514
</td>
<td style="text-align:right;">
-0.068390
</td>
<td style="text-align:right;">
0.030864
</td>
<td style="text-align:right;">
-0.026487
</td>
<td style="text-align:right;">
-0.413846
</td>
<td style="text-align:right;">
-0.140958
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.240714
</td>
<td style="text-align:right;">
-0.216972
</td>
<td style="text-align:right;">
-0.037211
</td>
<td style="text-align:right;">
0.026265
</td>
<td style="text-align:right;">
-0.017630
</td>
<td style="text-align:right;">
-0.032130
</td>
<td style="text-align:right;">
0.001049
</td>
<td style="text-align:right;">
-0.016263
</td>
<td style="text-align:right;">
0.003139
</td>
<td style="text-align:right;">
-0.047196
</td>
<td style="text-align:right;">
0.025594
</td>
<td style="text-align:right;">
0.071818
</td>
<td style="text-align:right;">
0.004399
</td>
<td style="text-align:right;">
0.060440
</td>
<td style="text-align:right;">
0.000812
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_03
</td>
<td style="text-align:right;">
-0.031840
</td>
<td style="text-align:right;">
-0.098840
</td>
<td style="text-align:right;">
0.193896
</td>
<td style="text-align:right;">
0.173339
</td>
<td style="text-align:right;">
-0.015255
</td>
<td style="text-align:right;">
-0.113095
</td>
<td style="text-align:right;">
0.011806
</td>
<td style="text-align:right;">
0.162094
</td>
<td style="text-align:right;">
-0.064123
</td>
<td style="text-align:right;">
0.042726
</td>
<td style="text-align:right;">
-0.019037
</td>
<td style="text-align:right;">
0.015102
</td>
<td style="text-align:right;">
0.009839
</td>
<td style="text-align:right;">
0.018556
</td>
<td style="text-align:right;">
-0.030290
</td>
<td style="text-align:right;">
0.119711
</td>
<td style="text-align:right;">
0.022436
</td>
<td style="text-align:right;">
0.128448
</td>
<td style="text-align:right;">
0.189739
</td>
<td style="text-align:right;">
-0.026244
</td>
<td style="text-align:right;">
-0.068364
</td>
<td style="text-align:right;">
-0.049053
</td>
<td style="text-align:right;">
-0.495824
</td>
<td style="text-align:right;">
-0.056717
</td>
<td style="text-align:right;">
-0.240714
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.212647
</td>
<td style="text-align:right;">
0.176352
</td>
<td style="text-align:right;">
0.031158
</td>
<td style="text-align:right;">
0.024271
</td>
<td style="text-align:right;">
0.097966
</td>
<td style="text-align:right;">
-0.071314
</td>
<td style="text-align:right;">
0.064928
</td>
<td style="text-align:right;">
0.145163
</td>
<td style="text-align:right;">
0.148600
</td>
<td style="text-align:right;">
0.013722
</td>
<td style="text-align:right;">
-0.075323
</td>
<td style="text-align:right;">
-0.034996
</td>
<td style="text-align:right;">
-0.005537
</td>
<td style="text-align:right;">
0.139432
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_04
</td>
<td style="text-align:right;">
0.012507
</td>
<td style="text-align:right;">
0.030998
</td>
<td style="text-align:right;">
-0.080025
</td>
<td style="text-align:right;">
-0.058351
</td>
<td style="text-align:right;">
-0.083454
</td>
<td style="text-align:right;">
0.080542
</td>
<td style="text-align:right;">
-0.051641
</td>
<td style="text-align:right;">
-0.053046
</td>
<td style="text-align:right;">
-0.015882
</td>
<td style="text-align:right;">
0.077732
</td>
<td style="text-align:right;">
0.018961
</td>
<td style="text-align:right;">
-0.017233
</td>
<td style="text-align:right;">
-0.021197
</td>
<td style="text-align:right;">
0.033226
</td>
<td style="text-align:right;">
-0.007801
</td>
<td style="text-align:right;">
-0.023578
</td>
<td style="text-align:right;">
0.037051
</td>
<td style="text-align:right;">
-0.064459
</td>
<td style="text-align:right;">
-0.071381
</td>
<td style="text-align:right;">
-0.029323
</td>
<td style="text-align:right;">
0.005978
</td>
<td style="text-align:right;">
-0.024722
</td>
<td style="text-align:right;">
-0.239921
</td>
<td style="text-align:right;">
-0.135426
</td>
<td style="text-align:right;">
-0.216972
</td>
<td style="text-align:right;">
-0.212647
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.016254
</td>
<td style="text-align:right;">
0.033023
</td>
<td style="text-align:right;">
-0.036725
</td>
<td style="text-align:right;">
-0.103372
</td>
<td style="text-align:right;">
0.067705
</td>
<td style="text-align:right;">
-0.052949
</td>
<td style="text-align:right;">
-0.013524
</td>
<td style="text-align:right;">
0.014807
</td>
<td style="text-align:right;">
0.008605
</td>
<td style="text-align:right;">
0.054841
</td>
<td style="text-align:right;">
0.043956
</td>
<td style="text-align:right;">
0.025550
</td>
<td style="text-align:right;">
-0.050630
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Subj
</td>
<td style="text-align:right;">
-0.042673
</td>
<td style="text-align:right;">
0.111615
</td>
<td style="text-align:right;">
0.148311
</td>
<td style="text-align:right;">
0.214363
</td>
<td style="text-align:right;">
0.161788
</td>
<td style="text-align:right;">
0.055771
</td>
<td style="text-align:right;">
0.131823
</td>
<td style="text-align:right;">
0.143973
</td>
<td style="text-align:right;">
0.230634
</td>
<td style="text-align:right;">
0.023737
</td>
<td style="text-align:right;">
-0.054772
</td>
<td style="text-align:right;">
-0.015499
</td>
<td style="text-align:right;">
-0.021655
</td>
<td style="text-align:right;">
-0.057217
</td>
<td style="text-align:right;">
0.054425
</td>
<td style="text-align:right;">
0.000478
</td>
<td style="text-align:right;">
0.036209
</td>
<td style="text-align:right;">
-0.001842
</td>
<td style="text-align:right;">
0.043780
</td>
<td style="text-align:right;">
0.018144
</td>
<td style="text-align:right;">
0.005552
</td>
<td style="text-align:right;">
0.004360
</td>
<td style="text-align:right;">
-0.116692
</td>
<td style="text-align:right;">
0.015470
</td>
<td style="text-align:right;">
-0.037211
</td>
<td style="text-align:right;">
0.176352
</td>
<td style="text-align:right;">
-0.016254
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.308769
</td>
<td style="text-align:right;">
0.259346
</td>
<td style="text-align:right;">
0.190574
</td>
<td style="text-align:right;">
0.145544
</td>
<td style="text-align:right;">
0.045002
</td>
<td style="text-align:right;">
0.568682
</td>
<td style="text-align:right;">
0.238337
</td>
<td style="text-align:right;">
0.340609
</td>
<td style="text-align:right;">
-0.304297
</td>
<td style="text-align:right;">
-0.278754
</td>
<td style="text-align:right;">
-0.095063
</td>
<td style="text-align:right;">
0.173601
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pol
</td>
<td style="text-align:right;">
-0.019413
</td>
<td style="text-align:right;">
0.000062
</td>
<td style="text-align:right;">
0.041039
</td>
<td style="text-align:right;">
0.074294
</td>
<td style="text-align:right;">
0.080273
</td>
<td style="text-align:right;">
0.070478
</td>
<td style="text-align:right;">
0.067030
</td>
<td style="text-align:right;">
0.103311
</td>
<td style="text-align:right;">
0.082968
</td>
<td style="text-align:right;">
0.102337
</td>
<td style="text-align:right;">
0.020125
</td>
<td style="text-align:right;">
0.026132
</td>
<td style="text-align:right;">
0.024111
</td>
<td style="text-align:right;">
-0.049430
</td>
<td style="text-align:right;">
-0.009011
</td>
<td style="text-align:right;">
-0.092663
</td>
<td style="text-align:right;">
-0.057650
</td>
<td style="text-align:right;">
0.017160
</td>
<td style="text-align:right;">
0.007314
</td>
<td style="text-align:right;">
-0.010294
</td>
<td style="text-align:right;">
0.017693
</td>
<td style="text-align:right;">
-0.002259
</td>
<td style="text-align:right;">
-0.082915
</td>
<td style="text-align:right;">
0.029695
</td>
<td style="text-align:right;">
0.026265
</td>
<td style="text-align:right;">
0.031158
</td>
<td style="text-align:right;">
0.033023
</td>
<td style="text-align:right;">
0.308769
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.448492
</td>
<td style="text-align:right;">
-0.505430
</td>
<td style="text-align:right;">
0.677157
</td>
<td style="text-align:right;">
-0.661803
</td>
<td style="text-align:right;">
0.562423
</td>
<td style="text-align:right;">
0.130552
</td>
<td style="text-align:right;">
0.392775
</td>
<td style="text-align:right;">
0.277824
</td>
<td style="text-align:right;">
0.296206
</td>
<td style="text-align:right;">
0.015578
</td>
<td style="text-align:right;">
0.072826
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pos.Rate
</td>
<td style="text-align:right;">
-0.027215
</td>
<td style="text-align:right;">
0.084415
</td>
<td style="text-align:right;">
0.050837
</td>
<td style="text-align:right;">
0.091250
</td>
<td style="text-align:right;">
0.069826
</td>
<td style="text-align:right;">
0.133566
</td>
<td style="text-align:right;">
0.051219
</td>
<td style="text-align:right;">
0.095525
</td>
<td style="text-align:right;">
0.161050
</td>
<td style="text-align:right;">
0.082372
</td>
<td style="text-align:right;">
0.003283
</td>
<td style="text-align:right;">
0.007104
</td>
<td style="text-align:right;">
0.007055
</td>
<td style="text-align:right;">
-0.040781
</td>
<td style="text-align:right;">
0.009949
</td>
<td style="text-align:right;">
-0.045841
</td>
<td style="text-align:right;">
-0.027393
</td>
<td style="text-align:right;">
-0.015301
</td>
<td style="text-align:right;">
0.010108
</td>
<td style="text-align:right;">
0.008405
</td>
<td style="text-align:right;">
0.030309
</td>
<td style="text-align:right;">
0.003404
</td>
<td style="text-align:right;">
0.018877
</td>
<td style="text-align:right;">
-0.002271
</td>
<td style="text-align:right;">
-0.017630
</td>
<td style="text-align:right;">
0.024271
</td>
<td style="text-align:right;">
-0.036725
</td>
<td style="text-align:right;">
0.259346
</td>
<td style="text-align:right;">
0.448492
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.050780
</td>
<td style="text-align:right;">
0.488698
</td>
<td style="text-align:right;">
-0.408297
</td>
<td style="text-align:right;">
0.110211
</td>
<td style="text-align:right;">
-0.203001
</td>
<td style="text-align:right;">
0.297992
</td>
<td style="text-align:right;">
-0.049140
</td>
<td style="text-align:right;">
-0.062191
</td>
<td style="text-align:right;">
-0.022523
</td>
<td style="text-align:right;">
0.097420
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Neg.Rate
</td>
<td style="text-align:right;">
-0.003254
</td>
<td style="text-align:right;">
0.117882
</td>
<td style="text-align:right;">
0.043769
</td>
<td style="text-align:right;">
0.080587
</td>
<td style="text-align:right;">
0.096988
</td>
<td style="text-align:right;">
0.016976
</td>
<td style="text-align:right;">
0.071798
</td>
<td style="text-align:right;">
0.011801
</td>
<td style="text-align:right;">
0.070748
</td>
<td style="text-align:right;">
-0.072845
</td>
<td style="text-align:right;">
-0.035970
</td>
<td style="text-align:right;">
-0.002184
</td>
<td style="text-align:right;">
0.002663
</td>
<td style="text-align:right;">
0.019348
</td>
<td style="text-align:right;">
0.014265
</td>
<td style="text-align:right;">
0.062457
</td>
<td style="text-align:right;">
0.060514
</td>
<td style="text-align:right;">
0.002668
</td>
<td style="text-align:right;">
0.043646
</td>
<td style="text-align:right;">
0.020313
</td>
<td style="text-align:right;">
0.001111
</td>
<td style="text-align:right;">
0.004595
</td>
<td style="text-align:right;">
0.007736
</td>
<td style="text-align:right;">
0.006481
</td>
<td style="text-align:right;">
-0.032130
</td>
<td style="text-align:right;">
0.097966
</td>
<td style="text-align:right;">
-0.103372
</td>
<td style="text-align:right;">
0.190574
</td>
<td style="text-align:right;">
-0.505430
</td>
<td style="text-align:right;">
0.050780
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.677421
</td>
<td style="text-align:right;">
0.790745
</td>
<td style="text-align:right;">
0.065516
</td>
<td style="text-align:right;">
-0.036083
</td>
<td style="text-align:right;">
0.089794
</td>
<td style="text-align:right;">
-0.282001
</td>
<td style="text-align:right;">
-0.422820
</td>
<td style="text-align:right;">
0.083647
</td>
<td style="text-align:right;">
0.077375
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Pos
</td>
<td style="text-align:right;">
-0.014461
</td>
<td style="text-align:right;">
-0.038620
</td>
<td style="text-align:right;">
0.130234
</td>
<td style="text-align:right;">
0.150073
</td>
<td style="text-align:right;">
-0.027709
</td>
<td style="text-align:right;">
0.063227
</td>
<td style="text-align:right;">
-0.021570
</td>
<td style="text-align:right;">
0.027962
</td>
<td style="text-align:right;">
0.317581
</td>
<td style="text-align:right;">
0.088671
</td>
<td style="text-align:right;">
0.024376
</td>
<td style="text-align:right;">
0.012645
</td>
<td style="text-align:right;">
0.009106
</td>
<td style="text-align:right;">
-0.036565
</td>
<td style="text-align:right;">
0.003478
</td>
<td style="text-align:right;">
-0.065120
</td>
<td style="text-align:right;">
-0.047168
</td>
<td style="text-align:right;">
-0.016730
</td>
<td style="text-align:right;">
-0.031445
</td>
<td style="text-align:right;">
0.003641
</td>
<td style="text-align:right;">
0.037679
</td>
<td style="text-align:right;">
0.025854
</td>
<td style="text-align:right;">
0.016284
</td>
<td style="text-align:right;">
-0.001291
</td>
<td style="text-align:right;">
0.001049
</td>
<td style="text-align:right;">
-0.071314
</td>
<td style="text-align:right;">
0.067705
</td>
<td style="text-align:right;">
0.145544
</td>
<td style="text-align:right;">
0.677157
</td>
<td style="text-align:right;">
0.488698
</td>
<td style="text-align:right;">
-0.677421
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.860271
</td>
<td style="text-align:right;">
0.140600
</td>
<td style="text-align:right;">
-0.013974
</td>
<td style="text-align:right;">
0.182286
</td>
<td style="text-align:right;">
0.174801
</td>
<td style="text-align:right;">
0.292100
</td>
<td style="text-align:right;">
-0.096248
</td>
<td style="text-align:right;">
-0.005515
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Neg
</td>
<td style="text-align:right;">
0.004206
</td>
<td style="text-align:right;">
0.086445
</td>
<td style="text-align:right;">
0.040809
</td>
<td style="text-align:right;">
0.077240
</td>
<td style="text-align:right;">
0.065573
</td>
<td style="text-align:right;">
-0.036566
</td>
<td style="text-align:right;">
0.039967
</td>
<td style="text-align:right;">
-0.020238
</td>
<td style="text-align:right;">
0.107287
</td>
<td style="text-align:right;">
-0.092216
</td>
<td style="text-align:right;">
-0.044075
</td>
<td style="text-align:right;">
-0.013868
</td>
<td style="text-align:right;">
-0.010119
</td>
<td style="text-align:right;">
0.016700
</td>
<td style="text-align:right;">
0.011007
</td>
<td style="text-align:right;">
0.057692
</td>
<td style="text-align:right;">
0.066370
</td>
<td style="text-align:right;">
0.007991
</td>
<td style="text-align:right;">
0.024334
</td>
<td style="text-align:right;">
0.006165
</td>
<td style="text-align:right;">
-0.026934
</td>
<td style="text-align:right;">
-0.013936
</td>
<td style="text-align:right;">
-0.001099
</td>
<td style="text-align:right;">
-0.014781
</td>
<td style="text-align:right;">
-0.016263
</td>
<td style="text-align:right;">
0.064928
</td>
<td style="text-align:right;">
-0.052949
</td>
<td style="text-align:right;">
0.045002
</td>
<td style="text-align:right;">
-0.661803
</td>
<td style="text-align:right;">
-0.408297
</td>
<td style="text-align:right;">
0.790745
</td>
<td style="text-align:right;">
-0.860271
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.009764
</td>
<td style="text-align:right;">
0.059341
</td>
<td style="text-align:right;">
-0.048667
</td>
<td style="text-align:right;">
-0.269075
</td>
<td style="text-align:right;">
-0.386475
</td>
<td style="text-align:right;">
0.059942
</td>
<td style="text-align:right;">
0.012618
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Pos.Pol
</td>
<td style="text-align:right;">
-0.026661
</td>
<td style="text-align:right;">
0.102827
</td>
<td style="text-align:right;">
0.103214
</td>
<td style="text-align:right;">
0.189220
</td>
<td style="text-align:right;">
0.183652
</td>
<td style="text-align:right;">
0.047710
</td>
<td style="text-align:right;">
0.141501
</td>
<td style="text-align:right;">
0.128134
</td>
<td style="text-align:right;">
0.210778
</td>
<td style="text-align:right;">
0.050772
</td>
<td style="text-align:right;">
-0.032639
</td>
<td style="text-align:right;">
-0.009489
</td>
<td style="text-align:right;">
-0.012871
</td>
<td style="text-align:right;">
-0.029721
</td>
<td style="text-align:right;">
0.019560
</td>
<td style="text-align:right;">
-0.033693
</td>
<td style="text-align:right;">
0.013669
</td>
<td style="text-align:right;">
0.011598
</td>
<td style="text-align:right;">
0.032694
</td>
<td style="text-align:right;">
-0.008927
</td>
<td style="text-align:right;">
0.008033
</td>
<td style="text-align:right;">
-0.011644
</td>
<td style="text-align:right;">
-0.128565
</td>
<td style="text-align:right;">
0.023869
</td>
<td style="text-align:right;">
0.003139
</td>
<td style="text-align:right;">
0.145163
</td>
<td style="text-align:right;">
-0.013524
</td>
<td style="text-align:right;">
0.568682
</td>
<td style="text-align:right;">
0.562423
</td>
<td style="text-align:right;">
0.110211
</td>
<td style="text-align:right;">
0.065516
</td>
<td style="text-align:right;">
0.140600
</td>
<td style="text-align:right;">
0.009764
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.414634
</td>
<td style="text-align:right;">
0.586112
</td>
<td style="text-align:right;">
-0.105249
</td>
<td style="text-align:right;">
-0.119953
</td>
<td style="text-align:right;">
-0.031614
</td>
<td style="text-align:right;">
0.115361
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Pos.Pol
</td>
<td style="text-align:right;">
-0.061533
</td>
<td style="text-align:right;">
-0.281511
</td>
<td style="text-align:right;">
0.403185
</td>
<td style="text-align:right;">
0.338251
</td>
<td style="text-align:right;">
-0.175679
</td>
<td style="text-align:right;">
-0.152059
</td>
<td style="text-align:right;">
-0.059605
</td>
<td style="text-align:right;">
0.101776
</td>
<td style="text-align:right;">
0.045428
</td>
<td style="text-align:right;">
-0.138082
</td>
<td style="text-align:right;">
-0.037726
</td>
<td style="text-align:right;">
-0.026971
</td>
<td style="text-align:right;">
-0.028426
</td>
<td style="text-align:right;">
0.031287
</td>
<td style="text-align:right;">
0.019266
</td>
<td style="text-align:right;">
0.060646
</td>
<td style="text-align:right;">
0.065086
</td>
<td style="text-align:right;">
-0.008984
</td>
<td style="text-align:right;">
0.013512
</td>
<td style="text-align:right;">
-0.029373
</td>
<td style="text-align:right;">
-0.071943
</td>
<td style="text-align:right;">
-0.050623
</td>
<td style="text-align:right;">
-0.102799
</td>
<td style="text-align:right;">
0.010069
</td>
<td style="text-align:right;">
-0.047196
</td>
<td style="text-align:right;">
0.148600
</td>
<td style="text-align:right;">
0.014807
</td>
<td style="text-align:right;">
0.238337
</td>
<td style="text-align:right;">
0.130552
</td>
<td style="text-align:right;">
-0.203001
</td>
<td style="text-align:right;">
-0.036083
</td>
<td style="text-align:right;">
-0.013974
</td>
<td style="text-align:right;">
0.059341
</td>
<td style="text-align:right;">
0.414634
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.125707
</td>
<td style="text-align:right;">
0.019753
</td>
<td style="text-align:right;">
0.215002
</td>
<td style="text-align:right;">
-0.179494
</td>
<td style="text-align:right;">
0.049025
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Pos.Pol
</td>
<td style="text-align:right;">
0.018167
</td>
<td style="text-align:right;">
0.434432
</td>
<td style="text-align:right;">
-0.268433
</td>
<td style="text-align:right;">
-0.114314
</td>
<td style="text-align:right;">
0.343674
</td>
<td style="text-align:right;">
0.228867
</td>
<td style="text-align:right;">
0.212602
</td>
<td style="text-align:right;">
0.097821
</td>
<td style="text-align:right;">
0.201443
</td>
<td style="text-align:right;">
0.113028
</td>
<td style="text-align:right;">
-0.024010
</td>
<td style="text-align:right;">
0.039833
</td>
<td style="text-align:right;">
0.041878
</td>
<td style="text-align:right;">
-0.026794
</td>
<td style="text-align:right;">
0.021324
</td>
<td style="text-align:right;">
-0.062983
</td>
<td style="text-align:right;">
-0.017274
</td>
<td style="text-align:right;">
0.050004
</td>
<td style="text-align:right;">
0.047350
</td>
<td style="text-align:right;">
0.019369
</td>
<td style="text-align:right;">
0.073637
</td>
<td style="text-align:right;">
0.034292
</td>
<td style="text-align:right;">
-0.044806
</td>
<td style="text-align:right;">
0.014586
</td>
<td style="text-align:right;">
0.025594
</td>
<td style="text-align:right;">
0.013722
</td>
<td style="text-align:right;">
0.008605
</td>
<td style="text-align:right;">
0.340609
</td>
<td style="text-align:right;">
0.392775
</td>
<td style="text-align:right;">
0.297992
</td>
<td style="text-align:right;">
0.089794
</td>
<td style="text-align:right;">
0.182286
</td>
<td style="text-align:right;">
-0.048667
</td>
<td style="text-align:right;">
0.586112
</td>
<td style="text-align:right;">
-0.125707
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.148010
</td>
<td style="text-align:right;">
-0.338989
</td>
<td style="text-align:right;">
0.117779
</td>
<td style="text-align:right;">
0.065874
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Neg.Pol
</td>
<td style="text-align:right;">
-0.009660
</td>
<td style="text-align:right;">
-0.098763
</td>
<td style="text-align:right;">
-0.008291
</td>
<td style="text-align:right;">
-0.054773
</td>
<td style="text-align:right;">
-0.079859
</td>
<td style="text-align:right;">
-0.017300
</td>
<td style="text-align:right;">
-0.024237
</td>
<td style="text-align:right;">
-0.048857
</td>
<td style="text-align:right;">
-0.098138
</td>
<td style="text-align:right;">
-0.037944
</td>
<td style="text-align:right;">
0.031647
</td>
<td style="text-align:right;">
0.019775
</td>
<td style="text-align:right;">
0.026844
</td>
<td style="text-align:right;">
0.008521
</td>
<td style="text-align:right;">
-0.033717
</td>
<td style="text-align:right;">
-0.037725
</td>
<td style="text-align:right;">
-0.038474
</td>
<td style="text-align:right;">
-0.005933
</td>
<td style="text-align:right;">
-0.027477
</td>
<td style="text-align:right;">
-0.045621
</td>
<td style="text-align:right;">
-0.028245
</td>
<td style="text-align:right;">
-0.036237
</td>
<td style="text-align:right;">
-0.018766
</td>
<td style="text-align:right;">
-0.027742
</td>
<td style="text-align:right;">
0.071818
</td>
<td style="text-align:right;">
-0.075323
</td>
<td style="text-align:right;">
0.054841
</td>
<td style="text-align:right;">
-0.304297
</td>
<td style="text-align:right;">
0.277824
</td>
<td style="text-align:right;">
-0.049140
</td>
<td style="text-align:right;">
-0.282001
</td>
<td style="text-align:right;">
0.174801
</td>
<td style="text-align:right;">
-0.269075
</td>
<td style="text-align:right;">
-0.105249
</td>
<td style="text-align:right;">
0.019753
</td>
<td style="text-align:right;">
-0.148010
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.721615
</td>
<td style="text-align:right;">
0.624486
</td>
<td style="text-align:right;">
-0.075904
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Neg.Pol
</td>
<td style="text-align:right;">
-0.004838
</td>
<td style="text-align:right;">
-0.488994
</td>
<td style="text-align:right;">
0.328041
</td>
<td style="text-align:right;">
0.202848
</td>
<td style="text-align:right;">
-0.333871
</td>
<td style="text-align:right;">
-0.189858
</td>
<td style="text-align:right;">
-0.189987
</td>
<td style="text-align:right;">
-0.012029
</td>
<td style="text-align:right;">
-0.083225
</td>
<td style="text-align:right;">
-0.084582
</td>
<td style="text-align:right;">
0.019236
</td>
<td style="text-align:right;">
0.010582
</td>
<td style="text-align:right;">
0.019742
</td>
<td style="text-align:right;">
-0.015968
</td>
<td style="text-align:right;">
-0.014007
</td>
<td style="text-align:right;">
-0.003313
</td>
<td style="text-align:right;">
-0.018230
</td>
<td style="text-align:right;">
0.000746
</td>
<td style="text-align:right;">
-0.022950
</td>
<td style="text-align:right;">
-0.026125
</td>
<td style="text-align:right;">
-0.088865
</td>
<td style="text-align:right;">
-0.039568
</td>
<td style="text-align:right;">
-0.001788
</td>
<td style="text-align:right;">
-0.001795
</td>
<td style="text-align:right;">
0.004399
</td>
<td style="text-align:right;">
-0.034996
</td>
<td style="text-align:right;">
0.043956
</td>
<td style="text-align:right;">
-0.278754
</td>
<td style="text-align:right;">
0.296206
</td>
<td style="text-align:right;">
-0.062191
</td>
<td style="text-align:right;">
-0.422820
</td>
<td style="text-align:right;">
0.292100
</td>
<td style="text-align:right;">
-0.386475
</td>
<td style="text-align:right;">
-0.119953
</td>
<td style="text-align:right;">
0.215002
</td>
<td style="text-align:right;">
-0.338989
</td>
<td style="text-align:right;">
0.721615
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.068446
</td>
<td style="text-align:right;">
-0.073644
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Neg.Pol
</td>
<td style="text-align:right;">
0.023110
</td>
<td style="text-align:right;">
0.246887
</td>
<td style="text-align:right;">
-0.273257
</td>
<td style="text-align:right;">
-0.209937
</td>
<td style="text-align:right;">
0.153577
</td>
<td style="text-align:right;">
0.114013
</td>
<td style="text-align:right;">
0.093048
</td>
<td style="text-align:right;">
-0.028249
</td>
<td style="text-align:right;">
-0.059452
</td>
<td style="text-align:right;">
0.035014
</td>
<td style="text-align:right;">
0.006246
</td>
<td style="text-align:right;">
0.006722
</td>
<td style="text-align:right;">
0.005138
</td>
<td style="text-align:right;">
0.033554
</td>
<td style="text-align:right;">
-0.017239
</td>
<td style="text-align:right;">
-0.018002
</td>
<td style="text-align:right;">
-0.039292
</td>
<td style="text-align:right;">
-0.006929
</td>
<td style="text-align:right;">
-0.008937
</td>
<td style="text-align:right;">
-0.015282
</td>
<td style="text-align:right;">
0.026667
</td>
<td style="text-align:right;">
-0.005448
</td>
<td style="text-align:right;">
-0.047151
</td>
<td style="text-align:right;">
-0.034388
</td>
<td style="text-align:right;">
0.060440
</td>
<td style="text-align:right;">
-0.005537
</td>
<td style="text-align:right;">
0.025550
</td>
<td style="text-align:right;">
-0.095063
</td>
<td style="text-align:right;">
0.015578
</td>
<td style="text-align:right;">
-0.022523
</td>
<td style="text-align:right;">
0.083647
</td>
<td style="text-align:right;">
-0.096248
</td>
<td style="text-align:right;">
0.059942
</td>
<td style="text-align:right;">
-0.031614
</td>
<td style="text-align:right;">
-0.179494
</td>
<td style="text-align:right;">
0.117779
</td>
<td style="text-align:right;">
0.624486
</td>
<td style="text-align:right;">
0.068446
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.000097
</td>
</tr>
<tr>
<td style="text-align:left;">
Title.Subj
</td>
<td style="text-align:right;">
0.045058
</td>
<td style="text-align:right;">
0.035333
</td>
<td style="text-align:right;">
0.015635
</td>
<td style="text-align:right;">
0.008980
</td>
<td style="text-align:right;">
0.041272
</td>
<td style="text-align:right;">
-0.025830
</td>
<td style="text-align:right;">
0.084759
</td>
<td style="text-align:right;">
0.058564
</td>
<td style="text-align:right;">
-0.003999
</td>
<td style="text-align:right;">
-0.032221
</td>
<td style="text-align:right;">
0.003693
</td>
<td style="text-align:right;">
-0.010504
</td>
<td style="text-align:right;">
-0.010022
</td>
<td style="text-align:right;">
0.021113
</td>
<td style="text-align:right;">
-0.003558
</td>
<td style="text-align:right;">
0.016522
</td>
<td style="text-align:right;">
0.042713
</td>
<td style="text-align:right;">
0.004465
</td>
<td style="text-align:right;">
0.040571
</td>
<td style="text-align:right;">
-0.006702
</td>
<td style="text-align:right;">
0.010934
</td>
<td style="text-align:right;">
-0.004049
</td>
<td style="text-align:right;">
-0.102330
</td>
<td style="text-align:right;">
0.035696
</td>
<td style="text-align:right;">
0.000812
</td>
<td style="text-align:right;">
0.139432
</td>
<td style="text-align:right;">
-0.050630
</td>
<td style="text-align:right;">
0.173601
</td>
<td style="text-align:right;">
0.072826
</td>
<td style="text-align:right;">
0.097420
</td>
<td style="text-align:right;">
0.077375
</td>
<td style="text-align:right;">
-0.005515
</td>
<td style="text-align:right;">
0.012618
</td>
<td style="text-align:right;">
0.115361
</td>
<td style="text-align:right;">
0.049025
</td>
<td style="text-align:right;">
0.065874
</td>
<td style="text-align:right;">
-0.075904
</td>
<td style="text-align:right;">
-0.073644
</td>
<td style="text-align:right;">
0.000097
</td>
<td style="text-align:right;">
1.000000
</td>
</tr>
</tbody>
</table>

The above table gives the correlations between all variables in the
Social.Media data set. This allows us to see which two variables have
strong correlation. If we have two variables with a high correlation, we
might want to remove one of them to avoid too much multicollinearity.

``` r
#Correlation graph for lifestyle_train
correlation_graph(data_channel_train)
```

    ##                 Var1                Var2     value
    ## 82         n.Content         Rate.Unique -0.686668
    ## 122        n.Content Rate.Unique.Nonstop -0.561367
    ## 123      Rate.Unique Rate.Unique.Nonstop  0.926428
    ## 162        n.Content             n.Links  0.658262
    ## 205          n.Links             n.Other  0.624771
    ## 242        n.Content            n.Images  0.503601
    ## 245          n.Links            n.Images  0.581691
    ## 492    Max.Worst.Key       Avg.Worst.Key  0.978774
    ## 571    Min.Worst.Key        Max.Best.Key -0.855507
    ## 614     Min.Best.Key        Avg.Best.Key  0.665896
    ## 615     Max.Best.Key        Avg.Best.Key  0.539139
    ## 692    Max.Worst.Key         Avg.Max.Key  0.804291
    ## 693    Avg.Worst.Key         Avg.Max.Key  0.791916
    ## 732    Max.Worst.Key         Avg.Avg.Key  0.703510
    ## 733    Avg.Worst.Key         Avg.Avg.Key  0.679419
    ## 738      Avg.Max.Key         Avg.Avg.Key  0.853927
    ## 860          Min.Ref             Avg.Ref  0.820020
    ## 861          Max.Ref             Avg.Ref  0.780070
    ## 1229      Global.Pol     Global.Neg.Rate -0.505430
    ## 1269      Global.Pol            Rate.Pos  0.677157
    ## 1271 Global.Neg.Rate            Rate.Pos -0.677421
    ## 1309      Global.Pol            Rate.Neg -0.661803
    ## 1311 Global.Neg.Rate            Rate.Neg  0.790745
    ## 1312        Rate.Pos            Rate.Neg -0.860271
    ## 1348     Global.Subj         Avg.Pos.Pol  0.568682
    ## 1349      Global.Pol         Avg.Pos.Pol  0.562423
    ## 1434     Avg.Pos.Pol         Max.Pos.Pol  0.586112
    ## 1517     Avg.Neg.Pol         Min.Neg.Pol  0.721615
    ## 1557     Avg.Neg.Pol         Max.Neg.Pol  0.624486

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/r%20params$DataChannel%20corr_graph-1.png)<!-- -->

Because the correlation table above is large, it can be difficult to
read. The correlation graph above gives a visual summary of the table.
Using the legend, we are able to see the correlations between variables,
how strong the correlation is, and in what direction.

``` r
ggplot(shareshigh, aes(x=Rate.Pos, y=Rate.Neg,
                       color=Days_of_Week)) +
    geom_point(size=2)
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/scatterplot-1.png)<!-- -->

Once seeing the correlation table and graph, it is possible to graph two
variables on a scatterplot. This provides a visual of the linear
relationship. A scatterplot of two variables in the Social.Media dataset
has been created above.

``` r
## mean of shares 
mean(data_channel_train$shares)
```

    ## [1] 3658.17

``` r
## sd of shares 
sd(data_channel_train$shares)
```

    ## [1] 5160.17

``` r
## creates a new column that is if shares is higher than average or not 
shareshigh <- data_channel_train %>% select(shares) %>% mutate (shareshigh = (shares> mean(shares)))

## creates a contingency table of shareshigh and whether it is the weekend 
table(shareshigh$shareshigh, data_channel_train$Weekend)
```

    ##        
    ##            0    1
    ##   FALSE 1032  151
    ##   TRUE   377   66

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
    ##   Weekday 0.6346863 0.2318573
    ##   Weekend 0.0928659 0.0405904

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
    ##   Mon     0.0990160 0.0461255
    ##   Tues    0.1439114 0.0479705
    ##   Wed     0.1309963 0.0461255
    ##   Thurs   0.1537515 0.0485855
    ##   Fri     0.1070111 0.0430504
    ##   Weekend 0.0928659 0.0405904

After comparing shareshigh with whether or not the day was a weekend or
weekday, the above contingency table compares shareshigh for each
specific day of the week. Again, the frequencies are displayed as
relative frequencies.

``` r
ggplot(shareshigh, aes(x = Weekday, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Weekday or Weekend?') + 
  ylab('Relative Frequency')
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/weekday%20bar%20graph-1.png)<!-- -->

``` r
ggplot(shareshigh, aes(x = Days_of_Week, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Day of the Week') + 
  ylab('Relative Frequency')
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/day%20of%20the%20week%20graph-1.png)<!-- -->

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

    ## [1] " For Social.Media Thurs is the most frequent day of the week"

``` r
table(shareshigh$shareshigh, g$Most_Freq)
```

    ##        
    ##         Most Freq Day Not Most Freq Day
    ##   FALSE           250               933
    ##   TRUE             79               364

The above contingency table compares shareshigh to the Social.Media day
that occurs most frequently. This allows us to see if the most frequent
day tends to have more shareshigh.

``` r
## creates plotting object of shares
a <- ggplot(data_channel_train, aes(x=shares))

## histogram of shares 
a+geom_histogram(color= "red", fill="blue")+ ggtitle("Shares histogram")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/shares%20histogram-1.png)<!-- -->

Above we can see the frequency distribution of shares of the
Social.Media data channel. We should always see a long tail to the right
because a small number of articles will get a very high number of
shares. But looking at by looking at the distribution we can say how
many shares most of these articles got.

``` r
## creates plotting object with number of words in title and shares
b<- ggplot(data_channel_train, aes(x=n.Title, y=shares))

## creates a bar chart with number of words in title and shares 
b+ geom_col(fill="blue")+ ggtitle("Number of words in title vs shares") + labs(x="Number of words in title")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/col%20graph-1.png)<!-- -->

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

    ##         Rate.Unique         Min.Pos.Pol         Min.Neg.Pol Rate.Unique.Nonstop 
    ##        -0.096030483        -0.095645389        -0.091979004        -0.084416249 
    ##              LDA_02               Thurs        Min.Best.Key         Avg.Pos.Pol 
    ##        -0.077359077        -0.052985349        -0.048862283        -0.044166096 
    ##            Rate.Pos         Avg.Neg.Pol              LDA_01          Global.Pol 
    ##        -0.041521100        -0.041467908        -0.041054203        -0.041053802 
    ##            n.Images        Avg.Best.Key              LDA_03        Max.Best.Key 
    ##        -0.036955165        -0.035078539        -0.033738420        -0.033051932 
    ##           Avg.Words                Tues              LDA_04         Global.Subj 
    ##        -0.024838753        -0.023516604        -0.022549928        -0.021250073 
    ##             n.Other             n.Links     Global.Pos.Rate             n.Title 
    ##        -0.020706891        -0.013304546        -0.007731855        -0.004521997 
    ##        Rate.Nonstop                 Wed            n.Videos         Max.Pos.Pol 
    ##        -0.003640429        -0.001055730         0.000202414         0.002411495 
    ##                 Sat         Max.Neg.Pol            Abs.Subj     Global.Neg.Rate 
    ##         0.004465016         0.009293304         0.017576517         0.021100633 
    ##                 Mon               n.Key             Weekend                 Sun 
    ##         0.025798398         0.026418383         0.026756894         0.033779559 
    ##                 Fri       Min.Worst.Key            Rate.Neg           Title.Pol 
    ##         0.035738698         0.039767170         0.042563318         0.045816566 
    ##       Max.Worst.Key           n.Content       Avg.Worst.Key         Avg.Max.Key 
    ##         0.047537110         0.048565197         0.055082843         0.056535569 
    ##          Title.Subj             Max.Ref         Avg.Min.Key             Abs.Pol 
    ##         0.058840298         0.063661879         0.075409781         0.076458250 
    ##             Min.Ref             Avg.Ref         Avg.Avg.Key              LDA_00 
    ##         0.078808419         0.088491394         0.099677298         0.125202965 
    ##              shares 
    ##         1.000000000

``` r
## take the name of the highest correlated variable
highest_cor <-shares_correlations[52]  %>% names()

highest_cor
```

    ## [1] "LDA_00"

``` r
## creats scatter plot looking at shares vs highest correlated variable
g <-ggplot(data_channel_train,  aes(y=shares, x= data_channel_train[[highest_cor]])) 


g+ geom_point(aes(color=as.factor(Weekend))) +geom_smooth(method = lm) + ggtitle(" Highest correlated variable with shares") + labs(x="Highest correlated variable vs shares", color="Weekend")
```

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/graph%20of%20shares%20with%20highest%20correlated%20var-1.png)<!-- -->

The above graph looks at the relationship between shares and the
variable with the highest correlation for the Social.Media data channel,
and colored based on whether or not it is the weekend. because this is
the most positively correlated variable we should always see an upward
trend but the more correlated they are the more the dots will fall onto
the line of best fit.

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

![](C:/Users/Demetri/Documents/NCSU_masters/ST558/Repos/GitHub/ST_558_Project_2/Social.Media_files/figure-gfm/boosted%20tree%20tuning-1.png)<!-- -->

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
    ## RMSE       6189.23       6192.21 6187.88           6193.79

``` r
## gets the name of the column with the smallest rmse 
smallest_RMSE<-colnames(models_RMSE)[apply(models_RMSE,1,which.min)]

## declares the model with smallest RSME the winner 
paste0(" For ", 
        params$DataChannel, " ", 
       smallest_RMSE, " is the winner")
```

    ## [1] " For Social.Media rfRMSE is the winner"

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
