Project 2
================
Kristina Golden and Demetrios Samaras
2023-07-02

# World

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

In this report we will be looking at the World data channel of the
online news popularity data set. This data set looks at a wide range of
variables from 39644 different news articles. The response variable that
we will be focusing on is **shares**. The purpose of this analysis is to
try to predict how many shares a World article will get based on the
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

## World EDA

### World

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

## World Summarizations

``` r
#Shares table for data_channel_train
summary_table(data_channel_train)
```

    ##            Shares
    ## Minimum     35.00
    ## Q1         823.00
    ## Median    1100.00
    ## Q3        1800.00
    ## Maximum 284700.00
    ## Mean      2241.82
    ## SD        6047.67

The above table displays the World 5-number summary for the shares. It
also includes the mean and standard deviation. Because the mean is
greater than the median, we suspect that the World shares distribution
is right skewed.

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
0.054025
</td>
<td style="text-align:right;">
-0.058043
</td>
<td style="text-align:right;">
-0.038302
</td>
<td style="text-align:right;">
-0.002421
</td>
<td style="text-align:right;">
0.013074
</td>
<td style="text-align:right;">
-0.018913
</td>
<td style="text-align:right;">
0.031008
</td>
<td style="text-align:right;">
-0.029719
</td>
<td style="text-align:right;">
0.014503
</td>
<td style="text-align:right;">
-0.116090
</td>
<td style="text-align:right;">
-0.005757
</td>
<td style="text-align:right;">
-0.029859
</td>
<td style="text-align:right;">
0.015892
</td>
<td style="text-align:right;">
0.131274
</td>
<td style="text-align:right;">
0.101054
</td>
<td style="text-align:right;">
0.011272
</td>
<td style="text-align:right;">
0.003042
</td>
<td style="text-align:right;">
-0.001818
</td>
<td style="text-align:right;">
0.006697
</td>
<td style="text-align:right;">
0.001916
</td>
<td style="text-align:right;">
0.007686
</td>
<td style="text-align:right;">
-0.025425
</td>
<td style="text-align:right;">
0.029102
</td>
<td style="text-align:right;">
-0.038323
</td>
<td style="text-align:right;">
-0.020977
</td>
<td style="text-align:right;">
0.063274
</td>
<td style="text-align:right;">
-0.025519
</td>
<td style="text-align:right;">
-0.045439
</td>
<td style="text-align:right;">
-0.051152
</td>
<td style="text-align:right;">
0.004252
</td>
<td style="text-align:right;">
-0.022811
</td>
<td style="text-align:right;">
0.027096
</td>
<td style="text-align:right;">
-0.030281
</td>
<td style="text-align:right;">
-0.019376
</td>
<td style="text-align:right;">
-0.004459
</td>
<td style="text-align:right;">
-0.026539
</td>
<td style="text-align:right;">
-0.037666
</td>
<td style="text-align:right;">
-0.017548
</td>
<td style="text-align:right;">
0.060439
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Content
</td>
<td style="text-align:right;">
0.054025
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.309912
</td>
<td style="text-align:right;">
-0.151571
</td>
<td style="text-align:right;">
0.416770
</td>
<td style="text-align:right;">
0.171906
</td>
<td style="text-align:right;">
0.237710
</td>
<td style="text-align:right;">
0.056959
</td>
<td style="text-align:right;">
0.241221
</td>
<td style="text-align:right;">
0.083785
</td>
<td style="text-align:right;">
-0.086838
</td>
<td style="text-align:right;">
0.010314
</td>
<td style="text-align:right;">
-0.003476
</td>
<td style="text-align:right;">
-0.007190
</td>
<td style="text-align:right;">
0.099258
</td>
<td style="text-align:right;">
-0.012371
</td>
<td style="text-align:right;">
0.014263
</td>
<td style="text-align:right;">
0.004628
</td>
<td style="text-align:right;">
-0.011031
</td>
<td style="text-align:right;">
-0.014487
</td>
<td style="text-align:right;">
0.020144
</td>
<td style="text-align:right;">
-0.001270
</td>
<td style="text-align:right;">
-0.009569
</td>
<td style="text-align:right;">
-0.011369
</td>
<td style="text-align:right;">
0.052117
</td>
<td style="text-align:right;">
-0.103891
</td>
<td style="text-align:right;">
0.017708
</td>
<td style="text-align:right;">
0.191636
</td>
<td style="text-align:right;">
0.006612
</td>
<td style="text-align:right;">
0.139140
</td>
<td style="text-align:right;">
0.163618
</td>
<td style="text-align:right;">
0.119008
</td>
<td style="text-align:right;">
0.142783
</td>
<td style="text-align:right;">
0.166704
</td>
<td style="text-align:right;">
-0.212001
</td>
<td style="text-align:right;">
0.409047
</td>
<td style="text-align:right;">
-0.170995
</td>
<td style="text-align:right;">
-0.449132
</td>
<td style="text-align:right;">
0.174562
</td>
<td style="text-align:right;">
0.025754
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique
</td>
<td style="text-align:right;">
-0.058043
</td>
<td style="text-align:right;">
-0.309912
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.952860
</td>
<td style="text-align:right;">
-0.039431
</td>
<td style="text-align:right;">
0.053398
</td>
<td style="text-align:right;">
-0.377395
</td>
<td style="text-align:right;">
0.025451
</td>
<td style="text-align:right;">
0.722574
</td>
<td style="text-align:right;">
-0.076902
</td>
<td style="text-align:right;">
0.111782
</td>
<td style="text-align:right;">
-0.030027
</td>
<td style="text-align:right;">
-0.010644
</td>
<td style="text-align:right;">
0.002503
</td>
<td style="text-align:right;">
-0.126910
</td>
<td style="text-align:right;">
-0.006214
</td>
<td style="text-align:right;">
-0.029083
</td>
<td style="text-align:right;">
-0.055240
</td>
<td style="text-align:right;">
-0.055564
</td>
<td style="text-align:right;">
0.037081
</td>
<td style="text-align:right;">
0.032228
</td>
<td style="text-align:right;">
0.041913
</td>
<td style="text-align:right;">
0.045876
</td>
<td style="text-align:right;">
0.039492
</td>
<td style="text-align:right;">
-0.041346
</td>
<td style="text-align:right;">
0.003540
</td>
<td style="text-align:right;">
0.003583
</td>
<td style="text-align:right;">
0.512945
</td>
<td style="text-align:right;">
0.180015
</td>
<td style="text-align:right;">
0.317468
</td>
<td style="text-align:right;">
0.194941
</td>
<td style="text-align:right;">
0.493732
</td>
<td style="text-align:right;">
0.228917
</td>
<td style="text-align:right;">
0.473771
</td>
<td style="text-align:right;">
0.411623
</td>
<td style="text-align:right;">
0.230884
</td>
<td style="text-align:right;">
-0.259923
</td>
<td style="text-align:right;">
-0.039990
</td>
<td style="text-align:right;">
-0.329541
</td>
<td style="text-align:right;">
-0.029168
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Unique.Nonstop
</td>
<td style="text-align:right;">
-0.038302
</td>
<td style="text-align:right;">
-0.151571
</td>
<td style="text-align:right;">
0.952860
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.035444
</td>
<td style="text-align:right;">
0.079931
</td>
<td style="text-align:right;">
-0.412748
</td>
<td style="text-align:right;">
0.021227
</td>
<td style="text-align:right;">
0.773701
</td>
<td style="text-align:right;">
-0.057264
</td>
<td style="text-align:right;">
0.087059
</td>
<td style="text-align:right;">
-0.024498
</td>
<td style="text-align:right;">
-0.009635
</td>
<td style="text-align:right;">
-0.007319
</td>
<td style="text-align:right;">
-0.098001
</td>
<td style="text-align:right;">
-0.021569
</td>
<td style="text-align:right;">
-0.040296
</td>
<td style="text-align:right;">
-0.055634
</td>
<td style="text-align:right;">
-0.070573
</td>
<td style="text-align:right;">
0.038397
</td>
<td style="text-align:right;">
0.036656
</td>
<td style="text-align:right;">
0.043257
</td>
<td style="text-align:right;">
0.051831
</td>
<td style="text-align:right;">
0.047169
</td>
<td style="text-align:right;">
-0.040203
</td>
<td style="text-align:right;">
-0.047517
</td>
<td style="text-align:right;">
0.030274
</td>
<td style="text-align:right;">
0.583779
</td>
<td style="text-align:right;">
0.184483
</td>
<td style="text-align:right;">
0.377322
</td>
<td style="text-align:right;">
0.263984
</td>
<td style="text-align:right;">
0.525636
</td>
<td style="text-align:right;">
0.275630
</td>
<td style="text-align:right;">
0.539311
</td>
<td style="text-align:right;">
0.359382
</td>
<td style="text-align:right;">
0.342030
</td>
<td style="text-align:right;">
-0.320953
</td>
<td style="text-align:right;">
-0.158842
</td>
<td style="text-align:right;">
-0.287453
</td>
<td style="text-align:right;">
-0.030408
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Links
</td>
<td style="text-align:right;">
-0.002421
</td>
<td style="text-align:right;">
0.416770
</td>
<td style="text-align:right;">
-0.039431
</td>
<td style="text-align:right;">
-0.035444
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.267461
</td>
<td style="text-align:right;">
0.137090
</td>
<td style="text-align:right;">
0.105130
</td>
<td style="text-align:right;">
0.232290
</td>
<td style="text-align:right;">
0.077668
</td>
<td style="text-align:right;">
-0.049657
</td>
<td style="text-align:right;">
0.003140
</td>
<td style="text-align:right;">
-0.002175
</td>
<td style="text-align:right;">
-0.004114
</td>
<td style="text-align:right;">
0.053963
</td>
<td style="text-align:right;">
0.013510
</td>
<td style="text-align:right;">
0.011545
</td>
<td style="text-align:right;">
0.013908
</td>
<td style="text-align:right;">
0.030282
</td>
<td style="text-align:right;">
-0.014610
</td>
<td style="text-align:right;">
0.047308
</td>
<td style="text-align:right;">
0.012222
</td>
<td style="text-align:right;">
0.023349
</td>
<td style="text-align:right;">
-0.027775
</td>
<td style="text-align:right;">
-0.018146
</td>
<td style="text-align:right;">
0.038494
</td>
<td style="text-align:right;">
-0.004384
</td>
<td style="text-align:right;">
0.168585
</td>
<td style="text-align:right;">
0.059815
</td>
<td style="text-align:right;">
0.061655
</td>
<td style="text-align:right;">
0.025724
</td>
<td style="text-align:right;">
0.128253
</td>
<td style="text-align:right;">
0.061468
</td>
<td style="text-align:right;">
0.147247
</td>
<td style="text-align:right;">
-0.110680
</td>
<td style="text-align:right;">
0.258263
</td>
<td style="text-align:right;">
-0.155245
</td>
<td style="text-align:right;">
-0.243975
</td>
<td style="text-align:right;">
0.032512
</td>
<td style="text-align:right;">
0.018495
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Other
</td>
<td style="text-align:right;">
0.013074
</td>
<td style="text-align:right;">
0.171906
</td>
<td style="text-align:right;">
0.053398
</td>
<td style="text-align:right;">
0.079931
</td>
<td style="text-align:right;">
0.267461
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.077621
</td>
<td style="text-align:right;">
0.069925
</td>
<td style="text-align:right;">
0.174535
</td>
<td style="text-align:right;">
0.118703
</td>
<td style="text-align:right;">
-0.023762
</td>
<td style="text-align:right;">
-0.024793
</td>
<td style="text-align:right;">
-0.027128
</td>
<td style="text-align:right;">
-0.025547
</td>
<td style="text-align:right;">
0.027865
</td>
<td style="text-align:right;">
0.019219
</td>
<td style="text-align:right;">
0.080095
</td>
<td style="text-align:right;">
-0.025901
</td>
<td style="text-align:right;">
0.005832
</td>
<td style="text-align:right;">
-0.023201
</td>
<td style="text-align:right;">
0.140124
</td>
<td style="text-align:right;">
0.040300
</td>
<td style="text-align:right;">
0.032291
</td>
<td style="text-align:right;">
0.001059
</td>
<td style="text-align:right;">
0.027112
</td>
<td style="text-align:right;">
0.006302
</td>
<td style="text-align:right;">
-0.057495
</td>
<td style="text-align:right;">
0.075902
</td>
<td style="text-align:right;">
0.051958
</td>
<td style="text-align:right;">
0.055191
</td>
<td style="text-align:right;">
-0.000547
</td>
<td style="text-align:right;">
0.128546
</td>
<td style="text-align:right;">
0.034521
</td>
<td style="text-align:right;">
0.113326
</td>
<td style="text-align:right;">
-0.034458
</td>
<td style="text-align:right;">
0.122925
</td>
<td style="text-align:right;">
-0.078497
</td>
<td style="text-align:right;">
-0.083680
</td>
<td style="text-align:right;">
-0.010238
</td>
<td style="text-align:right;">
-0.053466
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Images
</td>
<td style="text-align:right;">
-0.018913
</td>
<td style="text-align:right;">
0.237710
</td>
<td style="text-align:right;">
-0.377395
</td>
<td style="text-align:right;">
-0.412748
</td>
<td style="text-align:right;">
0.137090
</td>
<td style="text-align:right;">
0.077621
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.039401
</td>
<td style="text-align:right;">
-0.230346
</td>
<td style="text-align:right;">
0.071659
</td>
<td style="text-align:right;">
-0.075244
</td>
<td style="text-align:right;">
-0.003173
</td>
<td style="text-align:right;">
-0.011817
</td>
<td style="text-align:right;">
0.016270
</td>
<td style="text-align:right;">
0.082746
</td>
<td style="text-align:right;">
0.019279
</td>
<td style="text-align:right;">
0.033381
</td>
<td style="text-align:right;">
0.064415
</td>
<td style="text-align:right;">
0.111697
</td>
<td style="text-align:right;">
0.018338
</td>
<td style="text-align:right;">
0.059504
</td>
<td style="text-align:right;">
0.038868
</td>
<td style="text-align:right;">
-0.023817
</td>
<td style="text-align:right;">
-0.011325
</td>
<td style="text-align:right;">
-0.059534
</td>
<td style="text-align:right;">
0.178473
</td>
<td style="text-align:right;">
-0.029677
</td>
<td style="text-align:right;">
-0.184146
</td>
<td style="text-align:right;">
-0.020491
</td>
<td style="text-align:right;">
-0.139751
</td>
<td style="text-align:right;">
-0.139580
</td>
<td style="text-align:right;">
-0.129272
</td>
<td style="text-align:right;">
-0.110887
</td>
<td style="text-align:right;">
-0.139500
</td>
<td style="text-align:right;">
-0.133232
</td>
<td style="text-align:right;">
-0.033476
</td>
<td style="text-align:right;">
0.084411
</td>
<td style="text-align:right;">
-0.008683
</td>
<td style="text-align:right;">
0.097476
</td>
<td style="text-align:right;">
0.027299
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Videos
</td>
<td style="text-align:right;">
0.031008
</td>
<td style="text-align:right;">
0.056959
</td>
<td style="text-align:right;">
0.025451
</td>
<td style="text-align:right;">
0.021227
</td>
<td style="text-align:right;">
0.105130
</td>
<td style="text-align:right;">
0.069925
</td>
<td style="text-align:right;">
-0.039401
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.005200
</td>
<td style="text-align:right;">
-0.013295
</td>
<td style="text-align:right;">
-0.025609
</td>
<td style="text-align:right;">
-0.011432
</td>
<td style="text-align:right;">
-0.016888
</td>
<td style="text-align:right;">
-0.002811
</td>
<td style="text-align:right;">
0.029883
</td>
<td style="text-align:right;">
0.088366
</td>
<td style="text-align:right;">
0.028609
</td>
<td style="text-align:right;">
0.036845
</td>
<td style="text-align:right;">
0.071013
</td>
<td style="text-align:right;">
0.012734
</td>
<td style="text-align:right;">
0.065413
</td>
<td style="text-align:right;">
0.038446
</td>
<td style="text-align:right;">
-0.008723
</td>
<td style="text-align:right;">
-0.000821
</td>
<td style="text-align:right;">
-0.071285
</td>
<td style="text-align:right;">
0.156357
</td>
<td style="text-align:right;">
-0.013559
</td>
<td style="text-align:right;">
0.049092
</td>
<td style="text-align:right;">
0.015850
</td>
<td style="text-align:right;">
-0.000932
</td>
<td style="text-align:right;">
0.003240
</td>
<td style="text-align:right;">
0.008830
</td>
<td style="text-align:right;">
0.015098
</td>
<td style="text-align:right;">
0.054594
</td>
<td style="text-align:right;">
0.029902
</td>
<td style="text-align:right;">
0.054935
</td>
<td style="text-align:right;">
-0.036517
</td>
<td style="text-align:right;">
-0.046545
</td>
<td style="text-align:right;">
0.008559
</td>
<td style="text-align:right;">
0.048238
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Words
</td>
<td style="text-align:right;">
-0.029719
</td>
<td style="text-align:right;">
0.241221
</td>
<td style="text-align:right;">
0.722574
</td>
<td style="text-align:right;">
0.773701
</td>
<td style="text-align:right;">
0.232290
</td>
<td style="text-align:right;">
0.174535
</td>
<td style="text-align:right;">
-0.230346
</td>
<td style="text-align:right;">
0.005200
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.043632
</td>
<td style="text-align:right;">
0.036181
</td>
<td style="text-align:right;">
-0.023622
</td>
<td style="text-align:right;">
-0.014818
</td>
<td style="text-align:right;">
-0.006817
</td>
<td style="text-align:right;">
-0.044934
</td>
<td style="text-align:right;">
-0.022666
</td>
<td style="text-align:right;">
-0.021468
</td>
<td style="text-align:right;">
-0.068907
</td>
<td style="text-align:right;">
-0.096101
</td>
<td style="text-align:right;">
0.019646
</td>
<td style="text-align:right;">
0.028482
</td>
<td style="text-align:right;">
0.027227
</td>
<td style="text-align:right;">
0.004642
</td>
<td style="text-align:right;">
-0.001841
</td>
<td style="text-align:right;">
0.057916
</td>
<td style="text-align:right;">
-0.117955
</td>
<td style="text-align:right;">
0.007369
</td>
<td style="text-align:right;">
0.613298
</td>
<td style="text-align:right;">
0.128273
</td>
<td style="text-align:right;">
0.340793
</td>
<td style="text-align:right;">
0.314810
</td>
<td style="text-align:right;">
0.560313
</td>
<td style="text-align:right;">
0.394140
</td>
<td style="text-align:right;">
0.567451
</td>
<td style="text-align:right;">
0.247937
</td>
<td style="text-align:right;">
0.476786
</td>
<td style="text-align:right;">
-0.384764
</td>
<td style="text-align:right;">
-0.338076
</td>
<td style="text-align:right;">
-0.228459
</td>
<td style="text-align:right;">
-0.026901
</td>
</tr>
<tr>
<td style="text-align:left;">
n.Key
</td>
<td style="text-align:right;">
0.014503
</td>
<td style="text-align:right;">
0.083785
</td>
<td style="text-align:right;">
-0.076902
</td>
<td style="text-align:right;">
-0.057264
</td>
<td style="text-align:right;">
0.077668
</td>
<td style="text-align:right;">
0.118703
</td>
<td style="text-align:right;">
0.071659
</td>
<td style="text-align:right;">
-0.013295
</td>
<td style="text-align:right;">
-0.043632
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.015993
</td>
<td style="text-align:right;">
0.093385
</td>
<td style="text-align:right;">
0.103723
</td>
<td style="text-align:right;">
-0.383202
</td>
<td style="text-align:right;">
0.025791
</td>
<td style="text-align:right;">
-0.397108
</td>
<td style="text-align:right;">
-0.322514
</td>
<td style="text-align:right;">
0.107192
</td>
<td style="text-align:right;">
-0.010835
</td>
<td style="text-align:right;">
-0.021658
</td>
<td style="text-align:right;">
0.023666
</td>
<td style="text-align:right;">
0.001223
</td>
<td style="text-align:right;">
0.066092
</td>
<td style="text-align:right;">
-0.061575
</td>
<td style="text-align:right;">
-0.161640
</td>
<td style="text-align:right;">
0.046029
</td>
<td style="text-align:right;">
0.160407
</td>
<td style="text-align:right;">
0.041915
</td>
<td style="text-align:right;">
0.122002
</td>
<td style="text-align:right;">
0.123469
</td>
<td style="text-align:right;">
-0.014948
</td>
<td style="text-align:right;">
0.067664
</td>
<td style="text-align:right;">
-0.120381
</td>
<td style="text-align:right;">
0.044768
</td>
<td style="text-align:right;">
-0.057971
</td>
<td style="text-align:right;">
0.090460
</td>
<td style="text-align:right;">
0.059467
</td>
<td style="text-align:right;">
0.014868
</td>
<td style="text-align:right;">
0.071518
</td>
<td style="text-align:right;">
0.021694
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Worst.Key
</td>
<td style="text-align:right;">
-0.116090
</td>
<td style="text-align:right;">
-0.086838
</td>
<td style="text-align:right;">
0.111782
</td>
<td style="text-align:right;">
0.087059
</td>
<td style="text-align:right;">
-0.049657
</td>
<td style="text-align:right;">
-0.023762
</td>
<td style="text-align:right;">
-0.075244
</td>
<td style="text-align:right;">
-0.025609
</td>
<td style="text-align:right;">
0.036181
</td>
<td style="text-align:right;">
-0.015993
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.022985
</td>
<td style="text-align:right;">
0.136193
</td>
<td style="text-align:right;">
-0.070657
</td>
<td style="text-align:right;">
-0.878377
</td>
<td style="text-align:right;">
-0.483368
</td>
<td style="text-align:right;">
-0.097050
</td>
<td style="text-align:right;">
-0.050426
</td>
<td style="text-align:right;">
-0.121810
</td>
<td style="text-align:right;">
-0.020281
</td>
<td style="text-align:right;">
-0.016786
</td>
<td style="text-align:right;">
-0.019408
</td>
<td style="text-align:right;">
-0.002790
</td>
<td style="text-align:right;">
0.018374
</td>
<td style="text-align:right;">
0.026504
</td>
<td style="text-align:right;">
0.048547
</td>
<td style="text-align:right;">
-0.073641
</td>
<td style="text-align:right;">
0.054467
</td>
<td style="text-align:right;">
0.066848
</td>
<td style="text-align:right;">
0.115456
</td>
<td style="text-align:right;">
0.009485
</td>
<td style="text-align:right;">
0.084231
</td>
<td style="text-align:right;">
-0.053293
</td>
<td style="text-align:right;">
0.041245
</td>
<td style="text-align:right;">
0.056722
</td>
<td style="text-align:right;">
0.015266
</td>
<td style="text-align:right;">
0.005078
</td>
<td style="text-align:right;">
0.060960
</td>
<td style="text-align:right;">
-0.038181
</td>
<td style="text-align:right;">
-0.005762
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Worst.Key
</td>
<td style="text-align:right;">
-0.005757
</td>
<td style="text-align:right;">
0.010314
</td>
<td style="text-align:right;">
-0.030027
</td>
<td style="text-align:right;">
-0.024498
</td>
<td style="text-align:right;">
0.003140
</td>
<td style="text-align:right;">
-0.024793
</td>
<td style="text-align:right;">
-0.003173
</td>
<td style="text-align:right;">
-0.011432
</td>
<td style="text-align:right;">
-0.023622
</td>
<td style="text-align:right;">
0.093385
</td>
<td style="text-align:right;">
0.022985
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.962156
</td>
<td style="text-align:right;">
-0.068970
</td>
<td style="text-align:right;">
-0.017110
</td>
<td style="text-align:right;">
-0.075227
</td>
<td style="text-align:right;">
-0.011134
</td>
<td style="text-align:right;">
0.575449
</td>
<td style="text-align:right;">
0.426269
</td>
<td style="text-align:right;">
-0.000635
</td>
<td style="text-align:right;">
0.005452
</td>
<td style="text-align:right;">
0.001892
</td>
<td style="text-align:right;">
0.017761
</td>
<td style="text-align:right;">
0.013266
</td>
<td style="text-align:right;">
-0.023531
</td>
<td style="text-align:right;">
-0.006195
</td>
<td style="text-align:right;">
0.016949
</td>
<td style="text-align:right;">
0.013250
</td>
<td style="text-align:right;">
0.010547
</td>
<td style="text-align:right;">
0.014771
</td>
<td style="text-align:right;">
-0.003353
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
-0.020876
</td>
<td style="text-align:right;">
0.005725
</td>
<td style="text-align:right;">
-0.022650
</td>
<td style="text-align:right;">
0.017301
</td>
<td style="text-align:right;">
0.001618
</td>
<td style="text-align:right;">
-0.001844
</td>
<td style="text-align:right;">
0.009420
</td>
<td style="text-align:right;">
0.033982
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Worst.Key
</td>
<td style="text-align:right;">
-0.029859
</td>
<td style="text-align:right;">
-0.003476
</td>
<td style="text-align:right;">
-0.010644
</td>
<td style="text-align:right;">
-0.009635
</td>
<td style="text-align:right;">
-0.002175
</td>
<td style="text-align:right;">
-0.027128
</td>
<td style="text-align:right;">
-0.011817
</td>
<td style="text-align:right;">
-0.016888
</td>
<td style="text-align:right;">
-0.014818
</td>
<td style="text-align:right;">
0.103723
</td>
<td style="text-align:right;">
0.136193
</td>
<td style="text-align:right;">
0.962156
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.110819
</td>
<td style="text-align:right;">
-0.133814
</td>
<td style="text-align:right;">
-0.182160
</td>
<td style="text-align:right;">
-0.028805
</td>
<td style="text-align:right;">
0.547496
</td>
<td style="text-align:right;">
0.400291
</td>
<td style="text-align:right;">
-0.003174
</td>
<td style="text-align:right;">
0.006312
</td>
<td style="text-align:right;">
0.000918
</td>
<td style="text-align:right;">
0.027665
</td>
<td style="text-align:right;">
0.019259
</td>
<td style="text-align:right;">
-0.034849
</td>
<td style="text-align:right;">
-0.010330
</td>
<td style="text-align:right;">
0.025286
</td>
<td style="text-align:right;">
0.028342
</td>
<td style="text-align:right;">
0.020308
</td>
<td style="text-align:right;">
0.039253
</td>
<td style="text-align:right;">
0.006680
</td>
<td style="text-align:right;">
0.009298
</td>
<td style="text-align:right;">
-0.025769
</td>
<td style="text-align:right;">
0.012992
</td>
<td style="text-align:right;">
-0.015255
</td>
<td style="text-align:right;">
0.025625
</td>
<td style="text-align:right;">
-0.002229
</td>
<td style="text-align:right;">
-0.000636
</td>
<td style="text-align:right;">
0.004036
</td>
<td style="text-align:right;">
0.033270
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Best.Key
</td>
<td style="text-align:right;">
0.015892
</td>
<td style="text-align:right;">
-0.007190
</td>
<td style="text-align:right;">
0.002503
</td>
<td style="text-align:right;">
-0.007319
</td>
<td style="text-align:right;">
-0.004114
</td>
<td style="text-align:right;">
-0.025547
</td>
<td style="text-align:right;">
0.016270
</td>
<td style="text-align:right;">
-0.002811
</td>
<td style="text-align:right;">
-0.006817
</td>
<td style="text-align:right;">
-0.383202
</td>
<td style="text-align:right;">
-0.070657
</td>
<td style="text-align:right;">
-0.068970
</td>
<td style="text-align:right;">
-0.110819
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.074208
</td>
<td style="text-align:right;">
0.467572
</td>
<td style="text-align:right;">
0.506553
</td>
<td style="text-align:right;">
0.012287
</td>
<td style="text-align:right;">
0.223708
</td>
<td style="text-align:right;">
0.014271
</td>
<td style="text-align:right;">
0.014300
</td>
<td style="text-align:right;">
0.018340
</td>
<td style="text-align:right;">
0.010781
</td>
<td style="text-align:right;">
0.004752
</td>
<td style="text-align:right;">
0.016601
</td>
<td style="text-align:right;">
0.069416
</td>
<td style="text-align:right;">
-0.077150
</td>
<td style="text-align:right;">
-0.023640
</td>
<td style="text-align:right;">
-0.011214
</td>
<td style="text-align:right;">
-0.032210
</td>
<td style="text-align:right;">
-0.022520
</td>
<td style="text-align:right;">
-0.010632
</td>
<td style="text-align:right;">
0.009510
</td>
<td style="text-align:right;">
-0.020824
</td>
<td style="text-align:right;">
-0.015080
</td>
<td style="text-align:right;">
-0.009691
</td>
<td style="text-align:right;">
-0.006155
</td>
<td style="text-align:right;">
0.012987
</td>
<td style="text-align:right;">
-0.025606
</td>
<td style="text-align:right;">
0.011015
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Best.Key
</td>
<td style="text-align:right;">
0.131274
</td>
<td style="text-align:right;">
0.099258
</td>
<td style="text-align:right;">
-0.126910
</td>
<td style="text-align:right;">
-0.098001
</td>
<td style="text-align:right;">
0.053963
</td>
<td style="text-align:right;">
0.027865
</td>
<td style="text-align:right;">
0.082746
</td>
<td style="text-align:right;">
0.029883
</td>
<td style="text-align:right;">
-0.044934
</td>
<td style="text-align:right;">
0.025791
</td>
<td style="text-align:right;">
-0.878377
</td>
<td style="text-align:right;">
-0.017110
</td>
<td style="text-align:right;">
-0.133814
</td>
<td style="text-align:right;">
0.074208
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.539282
</td>
<td style="text-align:right;">
0.101560
</td>
<td style="text-align:right;">
0.071941
</td>
<td style="text-align:right;">
0.155944
</td>
<td style="text-align:right;">
0.015531
</td>
<td style="text-align:right;">
0.016300
</td>
<td style="text-align:right;">
0.019606
</td>
<td style="text-align:right;">
0.014818
</td>
<td style="text-align:right;">
0.005151
</td>
<td style="text-align:right;">
-0.047848
</td>
<td style="text-align:right;">
-0.053238
</td>
<td style="text-align:right;">
0.085299
</td>
<td style="text-align:right;">
-0.064260
</td>
<td style="text-align:right;">
-0.069283
</td>
<td style="text-align:right;">
-0.124073
</td>
<td style="text-align:right;">
-0.016411
</td>
<td style="text-align:right;">
-0.087921
</td>
<td style="text-align:right;">
0.050698
</td>
<td style="text-align:right;">
-0.040156
</td>
<td style="text-align:right;">
-0.061335
</td>
<td style="text-align:right;">
-0.012099
</td>
<td style="text-align:right;">
-0.004534
</td>
<td style="text-align:right;">
-0.063453
</td>
<td style="text-align:right;">
0.046722
</td>
<td style="text-align:right;">
0.014344
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Best.Key
</td>
<td style="text-align:right;">
0.101054
</td>
<td style="text-align:right;">
-0.012371
</td>
<td style="text-align:right;">
-0.006214
</td>
<td style="text-align:right;">
-0.021569
</td>
<td style="text-align:right;">
0.013510
</td>
<td style="text-align:right;">
0.019219
</td>
<td style="text-align:right;">
0.019279
</td>
<td style="text-align:right;">
0.088366
</td>
<td style="text-align:right;">
-0.022666
</td>
<td style="text-align:right;">
-0.397108
</td>
<td style="text-align:right;">
-0.483368
</td>
<td style="text-align:right;">
-0.075227
</td>
<td style="text-align:right;">
-0.182160
</td>
<td style="text-align:right;">
0.467572
</td>
<td style="text-align:right;">
0.539282
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.380611
</td>
<td style="text-align:right;">
0.056800
</td>
<td style="text-align:right;">
0.299354
</td>
<td style="text-align:right;">
0.055915
</td>
<td style="text-align:right;">
0.054176
</td>
<td style="text-align:right;">
0.062805
</td>
<td style="text-align:right;">
-0.005228
</td>
<td style="text-align:right;">
-0.001872
</td>
<td style="text-align:right;">
0.022558
</td>
<td style="text-align:right;">
0.156595
</td>
<td style="text-align:right;">
-0.132255
</td>
<td style="text-align:right;">
-0.040440
</td>
<td style="text-align:right;">
-0.042410
</td>
<td style="text-align:right;">
-0.105007
</td>
<td style="text-align:right;">
-0.047590
</td>
<td style="text-align:right;">
-0.038160
</td>
<td style="text-align:right;">
0.026516
</td>
<td style="text-align:right;">
-0.030385
</td>
<td style="text-align:right;">
-0.001425
</td>
<td style="text-align:right;">
-0.041560
</td>
<td style="text-align:right;">
-0.031675
</td>
<td style="text-align:right;">
-0.000532
</td>
<td style="text-align:right;">
-0.031093
</td>
<td style="text-align:right;">
0.012210
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Min.Key
</td>
<td style="text-align:right;">
0.011272
</td>
<td style="text-align:right;">
0.014263
</td>
<td style="text-align:right;">
-0.029083
</td>
<td style="text-align:right;">
-0.040296
</td>
<td style="text-align:right;">
0.011545
</td>
<td style="text-align:right;">
0.080095
</td>
<td style="text-align:right;">
0.033381
</td>
<td style="text-align:right;">
0.028609
</td>
<td style="text-align:right;">
-0.021468
</td>
<td style="text-align:right;">
-0.322514
</td>
<td style="text-align:right;">
-0.097050
</td>
<td style="text-align:right;">
-0.011134
</td>
<td style="text-align:right;">
-0.028805
</td>
<td style="text-align:right;">
0.506553
</td>
<td style="text-align:right;">
0.101560
</td>
<td style="text-align:right;">
0.380611
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.037875
</td>
<td style="text-align:right;">
0.405035
</td>
<td style="text-align:right;">
0.021712
</td>
<td style="text-align:right;">
0.041834
</td>
<td style="text-align:right;">
0.034753
</td>
<td style="text-align:right;">
0.004464
</td>
<td style="text-align:right;">
-0.003751
</td>
<td style="text-align:right;">
0.075880
</td>
<td style="text-align:right;">
0.035281
</td>
<td style="text-align:right;">
-0.119769
</td>
<td style="text-align:right;">
-0.050427
</td>
<td style="text-align:right;">
-0.029477
</td>
<td style="text-align:right;">
-0.039018
</td>
<td style="text-align:right;">
-0.025417
</td>
<td style="text-align:right;">
-0.034559
</td>
<td style="text-align:right;">
0.009344
</td>
<td style="text-align:right;">
-0.033352
</td>
<td style="text-align:right;">
-0.014934
</td>
<td style="text-align:right;">
-0.022386
</td>
<td style="text-align:right;">
0.011102
</td>
<td style="text-align:right;">
0.016292
</td>
<td style="text-align:right;">
-0.002535
</td>
<td style="text-align:right;">
0.005223
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Max.Key
</td>
<td style="text-align:right;">
0.003042
</td>
<td style="text-align:right;">
0.004628
</td>
<td style="text-align:right;">
-0.055240
</td>
<td style="text-align:right;">
-0.055634
</td>
<td style="text-align:right;">
0.013908
</td>
<td style="text-align:right;">
-0.025901
</td>
<td style="text-align:right;">
0.064415
</td>
<td style="text-align:right;">
0.036845
</td>
<td style="text-align:right;">
-0.068907
</td>
<td style="text-align:right;">
0.107192
</td>
<td style="text-align:right;">
-0.050426
</td>
<td style="text-align:right;">
0.575449
</td>
<td style="text-align:right;">
0.547496
</td>
<td style="text-align:right;">
0.012287
</td>
<td style="text-align:right;">
0.071941
</td>
<td style="text-align:right;">
0.056800
</td>
<td style="text-align:right;">
0.037875
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.826386
</td>
<td style="text-align:right;">
0.036855
</td>
<td style="text-align:right;">
0.063756
</td>
<td style="text-align:right;">
0.055952
</td>
<td style="text-align:right;">
0.046095
</td>
<td style="text-align:right;">
0.020732
</td>
<td style="text-align:right;">
-0.133399
</td>
<td style="text-align:right;">
0.137907
</td>
<td style="text-align:right;">
0.034387
</td>
<td style="text-align:right;">
0.004722
</td>
<td style="text-align:right;">
0.023862
</td>
<td style="text-align:right;">
0.000754
</td>
<td style="text-align:right;">
-0.024364
</td>
<td style="text-align:right;">
-0.016490
</td>
<td style="text-align:right;">
-0.041655
</td>
<td style="text-align:right;">
-0.001980
</td>
<td style="text-align:right;">
-0.033956
</td>
<td style="text-align:right;">
0.000369
</td>
<td style="text-align:right;">
0.019050
</td>
<td style="text-align:right;">
0.020274
</td>
<td style="text-align:right;">
0.023782
</td>
<td style="text-align:right;">
0.037930
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Avg.Key
</td>
<td style="text-align:right;">
-0.001818
</td>
<td style="text-align:right;">
-0.011031
</td>
<td style="text-align:right;">
-0.055564
</td>
<td style="text-align:right;">
-0.070573
</td>
<td style="text-align:right;">
0.030282
</td>
<td style="text-align:right;">
0.005832
</td>
<td style="text-align:right;">
0.111697
</td>
<td style="text-align:right;">
0.071013
</td>
<td style="text-align:right;">
-0.096101
</td>
<td style="text-align:right;">
-0.010835
</td>
<td style="text-align:right;">
-0.121810
</td>
<td style="text-align:right;">
0.426269
</td>
<td style="text-align:right;">
0.400291
</td>
<td style="text-align:right;">
0.223708
</td>
<td style="text-align:right;">
0.155944
</td>
<td style="text-align:right;">
0.299354
</td>
<td style="text-align:right;">
0.405035
</td>
<td style="text-align:right;">
0.826386
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.058441
</td>
<td style="text-align:right;">
0.094192
</td>
<td style="text-align:right;">
0.086180
</td>
<td style="text-align:right;">
0.071325
</td>
<td style="text-align:right;">
0.014175
</td>
<td style="text-align:right;">
-0.208959
</td>
<td style="text-align:right;">
0.270519
</td>
<td style="text-align:right;">
0.025456
</td>
<td style="text-align:right;">
-0.008055
</td>
<td style="text-align:right;">
0.044785
</td>
<td style="text-align:right;">
0.012895
</td>
<td style="text-align:right;">
-0.039996
</td>
<td style="text-align:right;">
-0.018365
</td>
<td style="text-align:right;">
-0.064954
</td>
<td style="text-align:right;">
-0.000655
</td>
<td style="text-align:right;">
-0.027331
</td>
<td style="text-align:right;">
0.005774
</td>
<td style="text-align:right;">
0.029593
</td>
<td style="text-align:right;">
0.043595
</td>
<td style="text-align:right;">
0.015628
</td>
<td style="text-align:right;">
0.044618
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Ref
</td>
<td style="text-align:right;">
0.006697
</td>
<td style="text-align:right;">
-0.014487
</td>
<td style="text-align:right;">
0.037081
</td>
<td style="text-align:right;">
0.038397
</td>
<td style="text-align:right;">
-0.014610
</td>
<td style="text-align:right;">
-0.023201
</td>
<td style="text-align:right;">
0.018338
</td>
<td style="text-align:right;">
0.012734
</td>
<td style="text-align:right;">
0.019646
</td>
<td style="text-align:right;">
-0.021658
</td>
<td style="text-align:right;">
-0.020281
</td>
<td style="text-align:right;">
-0.000635
</td>
<td style="text-align:right;">
-0.003174
</td>
<td style="text-align:right;">
0.014271
</td>
<td style="text-align:right;">
0.015531
</td>
<td style="text-align:right;">
0.055915
</td>
<td style="text-align:right;">
0.021712
</td>
<td style="text-align:right;">
0.036855
</td>
<td style="text-align:right;">
0.058441
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.550680
</td>
<td style="text-align:right;">
0.878055
</td>
<td style="text-align:right;">
0.023681
</td>
<td style="text-align:right;">
0.003645
</td>
<td style="text-align:right;">
-0.050679
</td>
<td style="text-align:right;">
0.039545
</td>
<td style="text-align:right;">
0.020338
</td>
<td style="text-align:right;">
0.042690
</td>
<td style="text-align:right;">
0.022934
</td>
<td style="text-align:right;">
-0.002579
</td>
<td style="text-align:right;">
-0.009114
</td>
<td style="text-align:right;">
0.028076
</td>
<td style="text-align:right;">
0.000353
</td>
<td style="text-align:right;">
0.031312
</td>
<td style="text-align:right;">
0.024293
</td>
<td style="text-align:right;">
0.009508
</td>
<td style="text-align:right;">
0.007582
</td>
<td style="text-align:right;">
0.006029
</td>
<td style="text-align:right;">
-0.006777
</td>
<td style="text-align:right;">
-0.016858
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Ref
</td>
<td style="text-align:right;">
0.001916
</td>
<td style="text-align:right;">
0.020144
</td>
<td style="text-align:right;">
0.032228
</td>
<td style="text-align:right;">
0.036656
</td>
<td style="text-align:right;">
0.047308
</td>
<td style="text-align:right;">
0.140124
</td>
<td style="text-align:right;">
0.059504
</td>
<td style="text-align:right;">
0.065413
</td>
<td style="text-align:right;">
0.028482
</td>
<td style="text-align:right;">
0.023666
</td>
<td style="text-align:right;">
-0.016786
</td>
<td style="text-align:right;">
0.005452
</td>
<td style="text-align:right;">
0.006312
</td>
<td style="text-align:right;">
0.014300
</td>
<td style="text-align:right;">
0.016300
</td>
<td style="text-align:right;">
0.054176
</td>
<td style="text-align:right;">
0.041834
</td>
<td style="text-align:right;">
0.063756
</td>
<td style="text-align:right;">
0.094192
</td>
<td style="text-align:right;">
0.550680
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.837630
</td>
<td style="text-align:right;">
0.015712
</td>
<td style="text-align:right;">
0.010526
</td>
<td style="text-align:right;">
-0.048131
</td>
<td style="text-align:right;">
0.061028
</td>
<td style="text-align:right;">
0.003719
</td>
<td style="text-align:right;">
0.046423
</td>
<td style="text-align:right;">
0.034147
</td>
<td style="text-align:right;">
0.017021
</td>
<td style="text-align:right;">
-0.003006
</td>
<td style="text-align:right;">
0.039662
</td>
<td style="text-align:right;">
-0.004980
</td>
<td style="text-align:right;">
0.044295
</td>
<td style="text-align:right;">
-0.001438
</td>
<td style="text-align:right;">
0.036714
</td>
<td style="text-align:right;">
-0.007402
</td>
<td style="text-align:right;">
-0.024638
</td>
<td style="text-align:right;">
-0.006429
</td>
<td style="text-align:right;">
-0.005620
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Ref
</td>
<td style="text-align:right;">
0.007686
</td>
<td style="text-align:right;">
-0.001270
</td>
<td style="text-align:right;">
0.041913
</td>
<td style="text-align:right;">
0.043257
</td>
<td style="text-align:right;">
0.012222
</td>
<td style="text-align:right;">
0.040300
</td>
<td style="text-align:right;">
0.038868
</td>
<td style="text-align:right;">
0.038446
</td>
<td style="text-align:right;">
0.027227
</td>
<td style="text-align:right;">
0.001223
</td>
<td style="text-align:right;">
-0.019408
</td>
<td style="text-align:right;">
0.001892
</td>
<td style="text-align:right;">
0.000918
</td>
<td style="text-align:right;">
0.018340
</td>
<td style="text-align:right;">
0.019606
</td>
<td style="text-align:right;">
0.062805
</td>
<td style="text-align:right;">
0.034753
</td>
<td style="text-align:right;">
0.055952
</td>
<td style="text-align:right;">
0.086180
</td>
<td style="text-align:right;">
0.878055
</td>
<td style="text-align:right;">
0.837630
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.022960
</td>
<td style="text-align:right;">
0.009715
</td>
<td style="text-align:right;">
-0.059459
</td>
<td style="text-align:right;">
0.058323
</td>
<td style="text-align:right;">
0.015864
</td>
<td style="text-align:right;">
0.053990
</td>
<td style="text-align:right;">
0.034375
</td>
<td style="text-align:right;">
0.004315
</td>
<td style="text-align:right;">
-0.007540
</td>
<td style="text-align:right;">
0.037621
</td>
<td style="text-align:right;">
-0.001366
</td>
<td style="text-align:right;">
0.045838
</td>
<td style="text-align:right;">
0.014958
</td>
<td style="text-align:right;">
0.022399
</td>
<td style="text-align:right;">
0.002974
</td>
<td style="text-align:right;">
-0.005810
</td>
<td style="text-align:right;">
-0.006800
</td>
<td style="text-align:right;">
-0.012498
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_00
</td>
<td style="text-align:right;">
-0.025425
</td>
<td style="text-align:right;">
-0.009569
</td>
<td style="text-align:right;">
0.045876
</td>
<td style="text-align:right;">
0.051831
</td>
<td style="text-align:right;">
0.023349
</td>
<td style="text-align:right;">
0.032291
</td>
<td style="text-align:right;">
-0.023817
</td>
<td style="text-align:right;">
-0.008723
</td>
<td style="text-align:right;">
0.004642
</td>
<td style="text-align:right;">
0.066092
</td>
<td style="text-align:right;">
-0.002790
</td>
<td style="text-align:right;">
0.017761
</td>
<td style="text-align:right;">
0.027665
</td>
<td style="text-align:right;">
0.010781
</td>
<td style="text-align:right;">
0.014818
</td>
<td style="text-align:right;">
-0.005228
</td>
<td style="text-align:right;">
0.004464
</td>
<td style="text-align:right;">
0.046095
</td>
<td style="text-align:right;">
0.071325
</td>
<td style="text-align:right;">
0.023681
</td>
<td style="text-align:right;">
0.015712
</td>
<td style="text-align:right;">
0.022960
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.040442
</td>
<td style="text-align:right;">
-0.354679
</td>
<td style="text-align:right;">
-0.033081
</td>
<td style="text-align:right;">
-0.099350
</td>
<td style="text-align:right;">
0.027110
</td>
<td style="text-align:right;">
0.067271
</td>
<td style="text-align:right;">
0.074631
</td>
<td style="text-align:right;">
-0.041389
</td>
<td style="text-align:right;">
0.069521
</td>
<td style="text-align:right;">
-0.067783
</td>
<td style="text-align:right;">
0.039607
</td>
<td style="text-align:right;">
-0.027926
</td>
<td style="text-align:right;">
0.034160
</td>
<td style="text-align:right;">
-0.012555
</td>
<td style="text-align:right;">
0.019039
</td>
<td style="text-align:right;">
-0.024216
</td>
<td style="text-align:right;">
-0.014704
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_01
</td>
<td style="text-align:right;">
0.029102
</td>
<td style="text-align:right;">
-0.011369
</td>
<td style="text-align:right;">
0.039492
</td>
<td style="text-align:right;">
0.047169
</td>
<td style="text-align:right;">
-0.027775
</td>
<td style="text-align:right;">
0.001059
</td>
<td style="text-align:right;">
-0.011325
</td>
<td style="text-align:right;">
-0.000821
</td>
<td style="text-align:right;">
-0.001841
</td>
<td style="text-align:right;">
-0.061575
</td>
<td style="text-align:right;">
0.018374
</td>
<td style="text-align:right;">
0.013266
</td>
<td style="text-align:right;">
0.019259
</td>
<td style="text-align:right;">
0.004752
</td>
<td style="text-align:right;">
0.005151
</td>
<td style="text-align:right;">
-0.001872
</td>
<td style="text-align:right;">
-0.003751
</td>
<td style="text-align:right;">
0.020732
</td>
<td style="text-align:right;">
0.014175
</td>
<td style="text-align:right;">
0.003645
</td>
<td style="text-align:right;">
0.010526
</td>
<td style="text-align:right;">
0.009715
</td>
<td style="text-align:right;">
-0.040442
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.263768
</td>
<td style="text-align:right;">
-0.016452
</td>
<td style="text-align:right;">
-0.110960
</td>
<td style="text-align:right;">
0.043693
</td>
<td style="text-align:right;">
0.018299
</td>
<td style="text-align:right;">
0.011401
</td>
<td style="text-align:right;">
-0.029443
</td>
<td style="text-align:right;">
0.031443
</td>
<td style="text-align:right;">
-0.013941
</td>
<td style="text-align:right;">
0.050729
</td>
<td style="text-align:right;">
0.019311
</td>
<td style="text-align:right;">
0.020563
</td>
<td style="text-align:right;">
-0.055339
</td>
<td style="text-align:right;">
-0.020646
</td>
<td style="text-align:right;">
-0.025664
</td>
<td style="text-align:right;">
0.011807
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_02
</td>
<td style="text-align:right;">
-0.038323
</td>
<td style="text-align:right;">
0.052117
</td>
<td style="text-align:right;">
-0.041346
</td>
<td style="text-align:right;">
-0.040203
</td>
<td style="text-align:right;">
-0.018146
</td>
<td style="text-align:right;">
0.027112
</td>
<td style="text-align:right;">
-0.059534
</td>
<td style="text-align:right;">
-0.071285
</td>
<td style="text-align:right;">
0.057916
</td>
<td style="text-align:right;">
-0.161640
</td>
<td style="text-align:right;">
0.026504
</td>
<td style="text-align:right;">
-0.023531
</td>
<td style="text-align:right;">
-0.034849
</td>
<td style="text-align:right;">
0.016601
</td>
<td style="text-align:right;">
-0.047848
</td>
<td style="text-align:right;">
0.022558
</td>
<td style="text-align:right;">
0.075880
</td>
<td style="text-align:right;">
-0.133399
</td>
<td style="text-align:right;">
-0.208959
</td>
<td style="text-align:right;">
-0.050679
</td>
<td style="text-align:right;">
-0.048131
</td>
<td style="text-align:right;">
-0.059459
</td>
<td style="text-align:right;">
-0.354679
</td>
<td style="text-align:right;">
-0.263768
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.411600
</td>
<td style="text-align:right;">
-0.630406
</td>
<td style="text-align:right;">
-0.100851
</td>
<td style="text-align:right;">
-0.100339
</td>
<td style="text-align:right;">
-0.082789
</td>
<td style="text-align:right;">
0.042652
</td>
<td style="text-align:right;">
-0.058618
</td>
<td style="text-align:right;">
0.089670
</td>
<td style="text-align:right;">
-0.070058
</td>
<td style="text-align:right;">
-0.018713
</td>
<td style="text-align:right;">
-0.063943
</td>
<td style="text-align:right;">
0.048241
</td>
<td style="text-align:right;">
-0.015811
</td>
<td style="text-align:right;">
0.039560
</td>
<td style="text-align:right;">
-0.048517
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_03
</td>
<td style="text-align:right;">
-0.020977
</td>
<td style="text-align:right;">
-0.103891
</td>
<td style="text-align:right;">
0.003540
</td>
<td style="text-align:right;">
-0.047517
</td>
<td style="text-align:right;">
0.038494
</td>
<td style="text-align:right;">
0.006302
</td>
<td style="text-align:right;">
0.178473
</td>
<td style="text-align:right;">
0.156357
</td>
<td style="text-align:right;">
-0.117955
</td>
<td style="text-align:right;">
0.046029
</td>
<td style="text-align:right;">
0.048547
</td>
<td style="text-align:right;">
-0.006195
</td>
<td style="text-align:right;">
-0.010330
</td>
<td style="text-align:right;">
0.069416
</td>
<td style="text-align:right;">
-0.053238
</td>
<td style="text-align:right;">
0.156595
</td>
<td style="text-align:right;">
0.035281
</td>
<td style="text-align:right;">
0.137907
</td>
<td style="text-align:right;">
0.270519
</td>
<td style="text-align:right;">
0.039545
</td>
<td style="text-align:right;">
0.061028
</td>
<td style="text-align:right;">
0.058323
</td>
<td style="text-align:right;">
-0.033081
</td>
<td style="text-align:right;">
-0.016452
</td>
<td style="text-align:right;">
-0.411600
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.149894
</td>
<td style="text-align:right;">
0.019034
</td>
<td style="text-align:right;">
0.077754
</td>
<td style="text-align:right;">
-0.009037
</td>
<td style="text-align:right;">
-0.076658
</td>
<td style="text-align:right;">
-0.006771
</td>
<td style="text-align:right;">
-0.095228
</td>
<td style="text-align:right;">
0.026946
</td>
<td style="text-align:right;">
0.055656
</td>
<td style="text-align:right;">
-0.017042
</td>
<td style="text-align:right;">
0.000962
</td>
<td style="text-align:right;">
0.059597
</td>
<td style="text-align:right;">
-0.017177
</td>
<td style="text-align:right;">
0.051193
</td>
</tr>
<tr>
<td style="text-align:left;">
LDA_04
</td>
<td style="text-align:right;">
0.063274
</td>
<td style="text-align:right;">
0.017708
</td>
<td style="text-align:right;">
0.003583
</td>
<td style="text-align:right;">
0.030274
</td>
<td style="text-align:right;">
-0.004384
</td>
<td style="text-align:right;">
-0.057495
</td>
<td style="text-align:right;">
-0.029677
</td>
<td style="text-align:right;">
-0.013559
</td>
<td style="text-align:right;">
0.007369
</td>
<td style="text-align:right;">
0.160407
</td>
<td style="text-align:right;">
-0.073641
</td>
<td style="text-align:right;">
0.016949
</td>
<td style="text-align:right;">
0.025286
</td>
<td style="text-align:right;">
-0.077150
</td>
<td style="text-align:right;">
0.085299
</td>
<td style="text-align:right;">
-0.132255
</td>
<td style="text-align:right;">
-0.119769
</td>
<td style="text-align:right;">
0.034387
</td>
<td style="text-align:right;">
0.025456
</td>
<td style="text-align:right;">
0.020338
</td>
<td style="text-align:right;">
0.003719
</td>
<td style="text-align:right;">
0.015864
</td>
<td style="text-align:right;">
-0.099350
</td>
<td style="text-align:right;">
-0.110960
</td>
<td style="text-align:right;">
-0.630406
</td>
<td style="text-align:right;">
-0.149894
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.076028
</td>
<td style="text-align:right;">
0.023485
</td>
<td style="text-align:right;">
0.060483
</td>
<td style="text-align:right;">
0.037862
</td>
<td style="text-align:right;">
0.022253
</td>
<td style="text-align:right;">
0.000105
</td>
<td style="text-align:right;">
0.021545
</td>
<td style="text-align:right;">
-0.007954
</td>
<td style="text-align:right;">
0.061781
</td>
<td style="text-align:right;">
-0.027206
</td>
<td style="text-align:right;">
-0.022728
</td>
<td style="text-align:right;">
-0.011148
</td>
<td style="text-align:right;">
0.028128
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Subj
</td>
<td style="text-align:right;">
-0.025519
</td>
<td style="text-align:right;">
0.191636
</td>
<td style="text-align:right;">
0.512945
</td>
<td style="text-align:right;">
0.583779
</td>
<td style="text-align:right;">
0.168585
</td>
<td style="text-align:right;">
0.075902
</td>
<td style="text-align:right;">
-0.184146
</td>
<td style="text-align:right;">
0.049092
</td>
<td style="text-align:right;">
0.613298
</td>
<td style="text-align:right;">
0.041915
</td>
<td style="text-align:right;">
0.054467
</td>
<td style="text-align:right;">
0.013250
</td>
<td style="text-align:right;">
0.028342
</td>
<td style="text-align:right;">
-0.023640
</td>
<td style="text-align:right;">
-0.064260
</td>
<td style="text-align:right;">
-0.040440
</td>
<td style="text-align:right;">
-0.050427
</td>
<td style="text-align:right;">
0.004722
</td>
<td style="text-align:right;">
-0.008055
</td>
<td style="text-align:right;">
0.042690
</td>
<td style="text-align:right;">
0.046423
</td>
<td style="text-align:right;">
0.053990
</td>
<td style="text-align:right;">
0.027110
</td>
<td style="text-align:right;">
0.043693
</td>
<td style="text-align:right;">
-0.100851
</td>
<td style="text-align:right;">
0.019034
</td>
<td style="text-align:right;">
0.076028
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.337390
</td>
<td style="text-align:right;">
0.540178
</td>
<td style="text-align:right;">
0.256390
</td>
<td style="text-align:right;">
0.548945
</td>
<td style="text-align:right;">
0.093663
</td>
<td style="text-align:right;">
0.606599
</td>
<td style="text-align:right;">
0.198420
</td>
<td style="text-align:right;">
0.532812
</td>
<td style="text-align:right;">
-0.453078
</td>
<td style="text-align:right;">
-0.367701
</td>
<td style="text-align:right;">
-0.219751
</td>
<td style="text-align:right;">
0.118207
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pol
</td>
<td style="text-align:right;">
-0.045439
</td>
<td style="text-align:right;">
0.006612
</td>
<td style="text-align:right;">
0.180015
</td>
<td style="text-align:right;">
0.184483
</td>
<td style="text-align:right;">
0.059815
</td>
<td style="text-align:right;">
0.051958
</td>
<td style="text-align:right;">
-0.020491
</td>
<td style="text-align:right;">
0.015850
</td>
<td style="text-align:right;">
0.128273
</td>
<td style="text-align:right;">
0.122002
</td>
<td style="text-align:right;">
0.066848
</td>
<td style="text-align:right;">
0.010547
</td>
<td style="text-align:right;">
0.020308
</td>
<td style="text-align:right;">
-0.011214
</td>
<td style="text-align:right;">
-0.069283
</td>
<td style="text-align:right;">
-0.042410
</td>
<td style="text-align:right;">
-0.029477
</td>
<td style="text-align:right;">
0.023862
</td>
<td style="text-align:right;">
0.044785
</td>
<td style="text-align:right;">
0.022934
</td>
<td style="text-align:right;">
0.034147
</td>
<td style="text-align:right;">
0.034375
</td>
<td style="text-align:right;">
0.067271
</td>
<td style="text-align:right;">
0.018299
</td>
<td style="text-align:right;">
-0.100339
</td>
<td style="text-align:right;">
0.077754
</td>
<td style="text-align:right;">
0.023485
</td>
<td style="text-align:right;">
0.337390
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.626584
</td>
<td style="text-align:right;">
-0.449070
</td>
<td style="text-align:right;">
0.737455
</td>
<td style="text-align:right;">
-0.667660
</td>
<td style="text-align:right;">
0.496671
</td>
<td style="text-align:right;">
0.061672
</td>
<td style="text-align:right;">
0.453196
</td>
<td style="text-align:right;">
0.218367
</td>
<td style="text-align:right;">
0.296269
</td>
<td style="text-align:right;">
-0.066646
</td>
<td style="text-align:right;">
0.029453
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Pos.Rate
</td>
<td style="text-align:right;">
-0.051152
</td>
<td style="text-align:right;">
0.139140
</td>
<td style="text-align:right;">
0.317468
</td>
<td style="text-align:right;">
0.377322
</td>
<td style="text-align:right;">
0.061655
</td>
<td style="text-align:right;">
0.055191
</td>
<td style="text-align:right;">
-0.139751
</td>
<td style="text-align:right;">
-0.000932
</td>
<td style="text-align:right;">
0.340793
</td>
<td style="text-align:right;">
0.123469
</td>
<td style="text-align:right;">
0.115456
</td>
<td style="text-align:right;">
0.014771
</td>
<td style="text-align:right;">
0.039253
</td>
<td style="text-align:right;">
-0.032210
</td>
<td style="text-align:right;">
-0.124073
</td>
<td style="text-align:right;">
-0.105007
</td>
<td style="text-align:right;">
-0.039018
</td>
<td style="text-align:right;">
0.000754
</td>
<td style="text-align:right;">
0.012895
</td>
<td style="text-align:right;">
-0.002579
</td>
<td style="text-align:right;">
0.017021
</td>
<td style="text-align:right;">
0.004315
</td>
<td style="text-align:right;">
0.074631
</td>
<td style="text-align:right;">
0.011401
</td>
<td style="text-align:right;">
-0.082789
</td>
<td style="text-align:right;">
-0.009037
</td>
<td style="text-align:right;">
0.060483
</td>
<td style="text-align:right;">
0.540178
</td>
<td style="text-align:right;">
0.626584
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.100884
</td>
<td style="text-align:right;">
0.685501
</td>
<td style="text-align:right;">
-0.359583
</td>
<td style="text-align:right;">
0.381911
</td>
<td style="text-align:right;">
-0.070747
</td>
<td style="text-align:right;">
0.519510
</td>
<td style="text-align:right;">
-0.116718
</td>
<td style="text-align:right;">
-0.107185
</td>
<td style="text-align:right;">
-0.064532
</td>
<td style="text-align:right;">
0.112153
</td>
</tr>
<tr>
<td style="text-align:left;">
Global.Neg.Rate
</td>
<td style="text-align:right;">
0.004252
</td>
<td style="text-align:right;">
0.163618
</td>
<td style="text-align:right;">
0.194941
</td>
<td style="text-align:right;">
0.263984
</td>
<td style="text-align:right;">
0.025724
</td>
<td style="text-align:right;">
-0.000547
</td>
<td style="text-align:right;">
-0.139580
</td>
<td style="text-align:right;">
0.003240
</td>
<td style="text-align:right;">
0.314810
</td>
<td style="text-align:right;">
-0.014948
</td>
<td style="text-align:right;">
0.009485
</td>
<td style="text-align:right;">
-0.003353
</td>
<td style="text-align:right;">
0.006680
</td>
<td style="text-align:right;">
-0.022520
</td>
<td style="text-align:right;">
-0.016411
</td>
<td style="text-align:right;">
-0.047590
</td>
<td style="text-align:right;">
-0.025417
</td>
<td style="text-align:right;">
-0.024364
</td>
<td style="text-align:right;">
-0.039996
</td>
<td style="text-align:right;">
-0.009114
</td>
<td style="text-align:right;">
-0.003006
</td>
<td style="text-align:right;">
-0.007540
</td>
<td style="text-align:right;">
-0.041389
</td>
<td style="text-align:right;">
-0.029443
</td>
<td style="text-align:right;">
0.042652
</td>
<td style="text-align:right;">
-0.076658
</td>
<td style="text-align:right;">
0.037862
</td>
<td style="text-align:right;">
0.256390
</td>
<td style="text-align:right;">
-0.449070
</td>
<td style="text-align:right;">
0.100884
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.356058
</td>
<td style="text-align:right;">
0.771492
</td>
<td style="text-align:right;">
0.167967
</td>
<td style="text-align:right;">
0.074746
</td>
<td style="text-align:right;">
0.153056
</td>
<td style="text-align:right;">
-0.243238
</td>
<td style="text-align:right;">
-0.448036
</td>
<td style="text-align:right;">
0.116096
</td>
<td style="text-align:right;">
0.079085
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Pos
</td>
<td style="text-align:right;">
-0.022811
</td>
<td style="text-align:right;">
0.119008
</td>
<td style="text-align:right;">
0.493732
</td>
<td style="text-align:right;">
0.525636
</td>
<td style="text-align:right;">
0.128253
</td>
<td style="text-align:right;">
0.128546
</td>
<td style="text-align:right;">
-0.129272
</td>
<td style="text-align:right;">
0.008830
</td>
<td style="text-align:right;">
0.560313
</td>
<td style="text-align:right;">
0.067664
</td>
<td style="text-align:right;">
0.084231
</td>
<td style="text-align:right;">
-0.002419
</td>
<td style="text-align:right;">
0.009298
</td>
<td style="text-align:right;">
-0.010632
</td>
<td style="text-align:right;">
-0.087921
</td>
<td style="text-align:right;">
-0.038160
</td>
<td style="text-align:right;">
-0.034559
</td>
<td style="text-align:right;">
-0.016490
</td>
<td style="text-align:right;">
-0.018365
</td>
<td style="text-align:right;">
0.028076
</td>
<td style="text-align:right;">
0.039662
</td>
<td style="text-align:right;">
0.037621
</td>
<td style="text-align:right;">
0.069521
</td>
<td style="text-align:right;">
0.031443
</td>
<td style="text-align:right;">
-0.058618
</td>
<td style="text-align:right;">
-0.006771
</td>
<td style="text-align:right;">
0.022253
</td>
<td style="text-align:right;">
0.548945
</td>
<td style="text-align:right;">
0.737455
</td>
<td style="text-align:right;">
0.685501
</td>
<td style="text-align:right;">
-0.356058
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.504564
</td>
<td style="text-align:right;">
0.481004
</td>
<td style="text-align:right;">
0.073596
</td>
<td style="text-align:right;">
0.507578
</td>
<td style="text-align:right;">
-0.135682
</td>
<td style="text-align:right;">
0.027035
</td>
<td style="text-align:right;">
-0.252954
</td>
<td style="text-align:right;">
0.002987
</td>
</tr>
<tr>
<td style="text-align:left;">
Rate.Neg
</td>
<td style="text-align:right;">
0.027096
</td>
<td style="text-align:right;">
0.142783
</td>
<td style="text-align:right;">
0.228917
</td>
<td style="text-align:right;">
0.275630
</td>
<td style="text-align:right;">
0.061468
</td>
<td style="text-align:right;">
0.034521
</td>
<td style="text-align:right;">
-0.110887
</td>
<td style="text-align:right;">
0.015098
</td>
<td style="text-align:right;">
0.394140
</td>
<td style="text-align:right;">
-0.120381
</td>
<td style="text-align:right;">
-0.053293
</td>
<td style="text-align:right;">
-0.020876
</td>
<td style="text-align:right;">
-0.025769
</td>
<td style="text-align:right;">
0.009510
</td>
<td style="text-align:right;">
0.050698
</td>
<td style="text-align:right;">
0.026516
</td>
<td style="text-align:right;">
0.009344
</td>
<td style="text-align:right;">
-0.041655
</td>
<td style="text-align:right;">
-0.064954
</td>
<td style="text-align:right;">
0.000353
</td>
<td style="text-align:right;">
-0.004980
</td>
<td style="text-align:right;">
-0.001366
</td>
<td style="text-align:right;">
-0.067783
</td>
<td style="text-align:right;">
-0.013941
</td>
<td style="text-align:right;">
0.089670
</td>
<td style="text-align:right;">
-0.095228
</td>
<td style="text-align:right;">
0.000105
</td>
<td style="text-align:right;">
0.093663
</td>
<td style="text-align:right;">
-0.667660
</td>
<td style="text-align:right;">
-0.359583
</td>
<td style="text-align:right;">
0.771492
</td>
<td style="text-align:right;">
-0.504564
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.109827
</td>
<td style="text-align:right;">
0.206187
</td>
<td style="text-align:right;">
-0.014716
</td>
<td style="text-align:right;">
-0.284351
</td>
<td style="text-align:right;">
-0.419431
</td>
<td style="text-align:right;">
0.037633
</td>
<td style="text-align:right;">
-0.018447
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Pos.Pol
</td>
<td style="text-align:right;">
-0.030281
</td>
<td style="text-align:right;">
0.166704
</td>
<td style="text-align:right;">
0.473771
</td>
<td style="text-align:right;">
0.539311
</td>
<td style="text-align:right;">
0.147247
</td>
<td style="text-align:right;">
0.113326
</td>
<td style="text-align:right;">
-0.139500
</td>
<td style="text-align:right;">
0.054594
</td>
<td style="text-align:right;">
0.567451
</td>
<td style="text-align:right;">
0.044768
</td>
<td style="text-align:right;">
0.041245
</td>
<td style="text-align:right;">
0.005725
</td>
<td style="text-align:right;">
0.012992
</td>
<td style="text-align:right;">
-0.020824
</td>
<td style="text-align:right;">
-0.040156
</td>
<td style="text-align:right;">
-0.030385
</td>
<td style="text-align:right;">
-0.033352
</td>
<td style="text-align:right;">
-0.001980
</td>
<td style="text-align:right;">
-0.000655
</td>
<td style="text-align:right;">
0.031312
</td>
<td style="text-align:right;">
0.044295
</td>
<td style="text-align:right;">
0.045838
</td>
<td style="text-align:right;">
0.039607
</td>
<td style="text-align:right;">
0.050729
</td>
<td style="text-align:right;">
-0.070058
</td>
<td style="text-align:right;">
0.026946
</td>
<td style="text-align:right;">
0.021545
</td>
<td style="text-align:right;">
0.606599
</td>
<td style="text-align:right;">
0.496671
</td>
<td style="text-align:right;">
0.381911
</td>
<td style="text-align:right;">
0.167967
</td>
<td style="text-align:right;">
0.481004
</td>
<td style="text-align:right;">
0.109827
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.433358
</td>
<td style="text-align:right;">
0.707232
</td>
<td style="text-align:right;">
-0.278685
</td>
<td style="text-align:right;">
-0.226401
</td>
<td style="text-align:right;">
-0.149793
</td>
<td style="text-align:right;">
0.026843
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Pos.Pol
</td>
<td style="text-align:right;">
-0.019376
</td>
<td style="text-align:right;">
-0.212001
</td>
<td style="text-align:right;">
0.411623
</td>
<td style="text-align:right;">
0.359382
</td>
<td style="text-align:right;">
-0.110680
</td>
<td style="text-align:right;">
-0.034458
</td>
<td style="text-align:right;">
-0.133232
</td>
<td style="text-align:right;">
0.029902
</td>
<td style="text-align:right;">
0.247937
</td>
<td style="text-align:right;">
-0.057971
</td>
<td style="text-align:right;">
0.056722
</td>
<td style="text-align:right;">
-0.022650
</td>
<td style="text-align:right;">
-0.015255
</td>
<td style="text-align:right;">
-0.015080
</td>
<td style="text-align:right;">
-0.061335
</td>
<td style="text-align:right;">
-0.001425
</td>
<td style="text-align:right;">
-0.014934
</td>
<td style="text-align:right;">
-0.033956
</td>
<td style="text-align:right;">
-0.027331
</td>
<td style="text-align:right;">
0.024293
</td>
<td style="text-align:right;">
-0.001438
</td>
<td style="text-align:right;">
0.014958
</td>
<td style="text-align:right;">
-0.027926
</td>
<td style="text-align:right;">
0.019311
</td>
<td style="text-align:right;">
-0.018713
</td>
<td style="text-align:right;">
0.055656
</td>
<td style="text-align:right;">
-0.007954
</td>
<td style="text-align:right;">
0.198420
</td>
<td style="text-align:right;">
0.061672
</td>
<td style="text-align:right;">
-0.070747
</td>
<td style="text-align:right;">
0.074746
</td>
<td style="text-align:right;">
0.073596
</td>
<td style="text-align:right;">
0.206187
</td>
<td style="text-align:right;">
0.433358
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.036555
</td>
<td style="text-align:right;">
-0.055558
</td>
<td style="text-align:right;">
0.063910
</td>
<td style="text-align:right;">
-0.152678
</td>
<td style="text-align:right;">
-0.019441
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Pos.Pol
</td>
<td style="text-align:right;">
-0.004459
</td>
<td style="text-align:right;">
0.409047
</td>
<td style="text-align:right;">
0.230884
</td>
<td style="text-align:right;">
0.342030
</td>
<td style="text-align:right;">
0.258263
</td>
<td style="text-align:right;">
0.122925
</td>
<td style="text-align:right;">
-0.033476
</td>
<td style="text-align:right;">
0.054935
</td>
<td style="text-align:right;">
0.476786
</td>
<td style="text-align:right;">
0.090460
</td>
<td style="text-align:right;">
0.015266
</td>
<td style="text-align:right;">
0.017301
</td>
<td style="text-align:right;">
0.025625
</td>
<td style="text-align:right;">
-0.009691
</td>
<td style="text-align:right;">
-0.012099
</td>
<td style="text-align:right;">
-0.041560
</td>
<td style="text-align:right;">
-0.022386
</td>
<td style="text-align:right;">
0.000369
</td>
<td style="text-align:right;">
0.005774
</td>
<td style="text-align:right;">
0.009508
</td>
<td style="text-align:right;">
0.036714
</td>
<td style="text-align:right;">
0.022399
</td>
<td style="text-align:right;">
0.034160
</td>
<td style="text-align:right;">
0.020563
</td>
<td style="text-align:right;">
-0.063943
</td>
<td style="text-align:right;">
-0.017042
</td>
<td style="text-align:right;">
0.061781
</td>
<td style="text-align:right;">
0.532812
</td>
<td style="text-align:right;">
0.453196
</td>
<td style="text-align:right;">
0.519510
</td>
<td style="text-align:right;">
0.153056
</td>
<td style="text-align:right;">
0.507578
</td>
<td style="text-align:right;">
-0.014716
</td>
<td style="text-align:right;">
0.707232
</td>
<td style="text-align:right;">
0.036555
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
-0.264592
</td>
<td style="text-align:right;">
-0.318928
</td>
<td style="text-align:right;">
-0.037863
</td>
<td style="text-align:right;">
0.066545
</td>
</tr>
<tr>
<td style="text-align:left;">
Avg.Neg.Pol
</td>
<td style="text-align:right;">
-0.026539
</td>
<td style="text-align:right;">
-0.170995
</td>
<td style="text-align:right;">
-0.259923
</td>
<td style="text-align:right;">
-0.320953
</td>
<td style="text-align:right;">
-0.155245
</td>
<td style="text-align:right;">
-0.078497
</td>
<td style="text-align:right;">
0.084411
</td>
<td style="text-align:right;">
-0.036517
</td>
<td style="text-align:right;">
-0.384764
</td>
<td style="text-align:right;">
0.059467
</td>
<td style="text-align:right;">
0.005078
</td>
<td style="text-align:right;">
0.001618
</td>
<td style="text-align:right;">
-0.002229
</td>
<td style="text-align:right;">
-0.006155
</td>
<td style="text-align:right;">
-0.004534
</td>
<td style="text-align:right;">
-0.031675
</td>
<td style="text-align:right;">
0.011102
</td>
<td style="text-align:right;">
0.019050
</td>
<td style="text-align:right;">
0.029593
</td>
<td style="text-align:right;">
0.007582
</td>
<td style="text-align:right;">
-0.007402
</td>
<td style="text-align:right;">
0.002974
</td>
<td style="text-align:right;">
-0.012555
</td>
<td style="text-align:right;">
-0.055339
</td>
<td style="text-align:right;">
0.048241
</td>
<td style="text-align:right;">
0.000962
</td>
<td style="text-align:right;">
-0.027206
</td>
<td style="text-align:right;">
-0.453078
</td>
<td style="text-align:right;">
0.218367
</td>
<td style="text-align:right;">
-0.116718
</td>
<td style="text-align:right;">
-0.243238
</td>
<td style="text-align:right;">
-0.135682
</td>
<td style="text-align:right;">
-0.284351
</td>
<td style="text-align:right;">
-0.278685
</td>
<td style="text-align:right;">
-0.055558
</td>
<td style="text-align:right;">
-0.264592
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.735290
</td>
<td style="text-align:right;">
0.521304
</td>
<td style="text-align:right;">
-0.041182
</td>
</tr>
<tr>
<td style="text-align:left;">
Min.Neg.Pol
</td>
<td style="text-align:right;">
-0.037666
</td>
<td style="text-align:right;">
-0.449132
</td>
<td style="text-align:right;">
-0.039990
</td>
<td style="text-align:right;">
-0.158842
</td>
<td style="text-align:right;">
-0.243975
</td>
<td style="text-align:right;">
-0.083680
</td>
<td style="text-align:right;">
-0.008683
</td>
<td style="text-align:right;">
-0.046545
</td>
<td style="text-align:right;">
-0.338076
</td>
<td style="text-align:right;">
0.014868
</td>
<td style="text-align:right;">
0.060960
</td>
<td style="text-align:right;">
-0.001844
</td>
<td style="text-align:right;">
-0.000636
</td>
<td style="text-align:right;">
0.012987
</td>
<td style="text-align:right;">
-0.063453
</td>
<td style="text-align:right;">
-0.000532
</td>
<td style="text-align:right;">
0.016292
</td>
<td style="text-align:right;">
0.020274
</td>
<td style="text-align:right;">
0.043595
</td>
<td style="text-align:right;">
0.006029
</td>
<td style="text-align:right;">
-0.024638
</td>
<td style="text-align:right;">
-0.005810
</td>
<td style="text-align:right;">
0.019039
</td>
<td style="text-align:right;">
-0.020646
</td>
<td style="text-align:right;">
-0.015811
</td>
<td style="text-align:right;">
0.059597
</td>
<td style="text-align:right;">
-0.022728
</td>
<td style="text-align:right;">
-0.367701
</td>
<td style="text-align:right;">
0.296269
</td>
<td style="text-align:right;">
-0.107185
</td>
<td style="text-align:right;">
-0.448036
</td>
<td style="text-align:right;">
0.027035
</td>
<td style="text-align:right;">
-0.419431
</td>
<td style="text-align:right;">
-0.226401
</td>
<td style="text-align:right;">
0.063910
</td>
<td style="text-align:right;">
-0.318928
</td>
<td style="text-align:right;">
0.735290
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.067866
</td>
<td style="text-align:right;">
-0.043778
</td>
</tr>
<tr>
<td style="text-align:left;">
Max.Neg.Pol
</td>
<td style="text-align:right;">
-0.017548
</td>
<td style="text-align:right;">
0.174562
</td>
<td style="text-align:right;">
-0.329541
</td>
<td style="text-align:right;">
-0.287453
</td>
<td style="text-align:right;">
0.032512
</td>
<td style="text-align:right;">
-0.010238
</td>
<td style="text-align:right;">
0.097476
</td>
<td style="text-align:right;">
0.008559
</td>
<td style="text-align:right;">
-0.228459
</td>
<td style="text-align:right;">
0.071518
</td>
<td style="text-align:right;">
-0.038181
</td>
<td style="text-align:right;">
0.009420
</td>
<td style="text-align:right;">
0.004036
</td>
<td style="text-align:right;">
-0.025606
</td>
<td style="text-align:right;">
0.046722
</td>
<td style="text-align:right;">
-0.031093
</td>
<td style="text-align:right;">
-0.002535
</td>
<td style="text-align:right;">
0.023782
</td>
<td style="text-align:right;">
0.015628
</td>
<td style="text-align:right;">
-0.006777
</td>
<td style="text-align:right;">
-0.006429
</td>
<td style="text-align:right;">
-0.006800
</td>
<td style="text-align:right;">
-0.024216
</td>
<td style="text-align:right;">
-0.025664
</td>
<td style="text-align:right;">
0.039560
</td>
<td style="text-align:right;">
-0.017177
</td>
<td style="text-align:right;">
-0.011148
</td>
<td style="text-align:right;">
-0.219751
</td>
<td style="text-align:right;">
-0.066646
</td>
<td style="text-align:right;">
-0.064532
</td>
<td style="text-align:right;">
0.116096
</td>
<td style="text-align:right;">
-0.252954
</td>
<td style="text-align:right;">
0.037633
</td>
<td style="text-align:right;">
-0.149793
</td>
<td style="text-align:right;">
-0.152678
</td>
<td style="text-align:right;">
-0.037863
</td>
<td style="text-align:right;">
0.521304
</td>
<td style="text-align:right;">
0.067866
</td>
<td style="text-align:right;">
1.000000
</td>
<td style="text-align:right;">
0.017767
</td>
</tr>
<tr>
<td style="text-align:left;">
Title.Subj
</td>
<td style="text-align:right;">
0.060439
</td>
<td style="text-align:right;">
0.025754
</td>
<td style="text-align:right;">
-0.029168
</td>
<td style="text-align:right;">
-0.030408
</td>
<td style="text-align:right;">
0.018495
</td>
<td style="text-align:right;">
-0.053466
</td>
<td style="text-align:right;">
0.027299
</td>
<td style="text-align:right;">
0.048238
</td>
<td style="text-align:right;">
-0.026901
</td>
<td style="text-align:right;">
0.021694
</td>
<td style="text-align:right;">
-0.005762
</td>
<td style="text-align:right;">
0.033982
</td>
<td style="text-align:right;">
0.033270
</td>
<td style="text-align:right;">
0.011015
</td>
<td style="text-align:right;">
0.014344
</td>
<td style="text-align:right;">
0.012210
</td>
<td style="text-align:right;">
0.005223
</td>
<td style="text-align:right;">
0.037930
</td>
<td style="text-align:right;">
0.044618
</td>
<td style="text-align:right;">
-0.016858
</td>
<td style="text-align:right;">
-0.005620
</td>
<td style="text-align:right;">
-0.012498
</td>
<td style="text-align:right;">
-0.014704
</td>
<td style="text-align:right;">
0.011807
</td>
<td style="text-align:right;">
-0.048517
</td>
<td style="text-align:right;">
0.051193
</td>
<td style="text-align:right;">
0.028128
</td>
<td style="text-align:right;">
0.118207
</td>
<td style="text-align:right;">
0.029453
</td>
<td style="text-align:right;">
0.112153
</td>
<td style="text-align:right;">
0.079085
</td>
<td style="text-align:right;">
0.002987
</td>
<td style="text-align:right;">
-0.018447
</td>
<td style="text-align:right;">
0.026843
</td>
<td style="text-align:right;">
-0.019441
</td>
<td style="text-align:right;">
0.066545
</td>
<td style="text-align:right;">
-0.041182
</td>
<td style="text-align:right;">
-0.043778
</td>
<td style="text-align:right;">
0.017767
</td>
<td style="text-align:right;">
1.000000
</td>
</tr>
</tbody>
</table>

The above table gives the correlations between all variables in the
World data set. This allows us to see which two variables have strong
correlation. If we have two variables with a high correlation, we might
want to remove one of them to avoid too much multicollinearity.

``` r
#Correlation graph for lifestyle_train
correlation_graph(data_channel_train)
```

    ##                     Var1                Var2     value
    ## 123          Rate.Unique Rate.Unique.Nonstop  0.952860
    ## 323          Rate.Unique           Avg.Words  0.722574
    ## 324  Rate.Unique.Nonstop           Avg.Words  0.773701
    ## 492        Max.Worst.Key       Avg.Worst.Key  0.962156
    ## 571        Min.Worst.Key        Max.Best.Key -0.878377
    ## 615         Max.Best.Key        Avg.Best.Key  0.539282
    ## 654         Min.Best.Key         Avg.Min.Key  0.506553
    ## 692        Max.Worst.Key         Avg.Max.Key  0.575449
    ## 693        Avg.Worst.Key         Avg.Max.Key  0.547496
    ## 738          Avg.Max.Key         Avg.Avg.Key  0.826386
    ## 820              Min.Ref             Max.Ref  0.550680
    ## 860              Min.Ref             Avg.Ref  0.878055
    ## 861              Max.Ref             Avg.Ref  0.837630
    ## 1065              LDA_02              LDA_04 -0.630406
    ## 1083         Rate.Unique         Global.Subj  0.512945
    ## 1084 Rate.Unique.Nonstop         Global.Subj  0.583779
    ## 1089           Avg.Words         Global.Subj  0.613298
    ## 1188         Global.Subj     Global.Pos.Rate  0.540178
    ## 1189          Global.Pol     Global.Pos.Rate  0.626584
    ## 1244 Rate.Unique.Nonstop            Rate.Pos  0.525636
    ## 1249           Avg.Words            Rate.Pos  0.560313
    ## 1268         Global.Subj            Rate.Pos  0.548945
    ## 1269          Global.Pol            Rate.Pos  0.737455
    ## 1270     Global.Pos.Rate            Rate.Pos  0.685501
    ## 1309          Global.Pol            Rate.Neg -0.667660
    ## 1311     Global.Neg.Rate            Rate.Neg  0.771492
    ## 1312            Rate.Pos            Rate.Neg -0.504564
    ## 1324 Rate.Unique.Nonstop         Avg.Pos.Pol  0.539311
    ## 1329           Avg.Words         Avg.Pos.Pol  0.567451
    ## 1348         Global.Subj         Avg.Pos.Pol  0.606599
    ## 1428         Global.Subj         Max.Pos.Pol  0.532812
    ## 1430     Global.Pos.Rate         Max.Pos.Pol  0.519510
    ## 1432            Rate.Pos         Max.Pos.Pol  0.507578
    ## 1434         Avg.Pos.Pol         Max.Pos.Pol  0.707232
    ## 1517         Avg.Neg.Pol         Min.Neg.Pol  0.735290
    ## 1557         Avg.Neg.Pol         Max.Neg.Pol  0.521304

![](World_files/figure-gfm/r%20params$DataChannel%20corr_graph-1.png)<!-- -->

Because the correlation table above is large, it can be difficult to
read. The correlation graph above gives a visual summary of the table.
Using the legend, we are able to see the correlations between variables,
how strong the correlation is, and in what direction.

``` r
ggplot(shareshigh, aes(x=Rate.Pos, y=Rate.Neg,
                       color=Days_of_Week)) +
    geom_point(size=2)
```

![](World_files/figure-gfm/scatterplot-1.png)<!-- -->

Once seeing the correlation table and graph, it is possible to graph two
variables on a scatterplot. This provides a visual of the linear
relationship. A scatterplot of two variables in the World dataset has
been created above.

``` r
## mean of shares 
mean(data_channel_train$shares)
```

    ## [1] 2241.82

``` r
## sd of shares 
sd(data_channel_train$shares)
```

    ## [1] 6047.67

``` r
## creates a new column that is if shares is higher than average or not 
shareshigh <- data_channel_train %>% select(shares) %>% mutate (shareshigh = (shares> mean(shares)))

## creates a contingency table of shareshigh and whether it is the weekend 
table(shareshigh$shareshigh, data_channel_train$Weekend)
```

    ##        
    ##            0    1
    ##   FALSE 4202  558
    ##   TRUE   934  204

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
    ##   Weekday 0.7124449 0.1583588
    ##   Weekend 0.0946083 0.0345880

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
    ##   Mon     0.1347915 0.0289929
    ##   Tues    0.1507291 0.0308579
    ##   Wed     0.1532723 0.0323839
    ##   Thurs   0.1485249 0.0366226
    ##   Fri     0.1251272 0.0295015
    ##   Weekend 0.0946083 0.0345880

After comparing shareshigh with whether or not the day was a weekend or
weekday, the above contingency table compares shareshigh for each
specific day of the week. Again, the frequencies are displayed as
relative frequencies.

``` r
ggplot(shareshigh, aes(x = Weekday, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Weekday or Weekend?') + 
  ylab('Relative Frequency')
```

![](World_files/figure-gfm/weekday%20bar%20graph-1.png)<!-- -->

``` r
ggplot(shareshigh, aes(x = Days_of_Week, fill = shareshigh)) +
  geom_bar(aes(y = (after_stat(count))/sum(after_stat(count)))) + xlab('Day of the Week') + 
  ylab('Relative Frequency')
```

![](World_files/figure-gfm/day%20of%20the%20week%20graph-1.png)<!-- -->

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

    ## [1] " For World Wed is the most frequent day of the week"

``` r
table(shareshigh$shareshigh, g$Most_Freq)
```

    ##        
    ##         Most Freq Day Not Most Freq Day
    ##   FALSE           904              3856
    ##   TRUE            191               947

The above contingency table compares shareshigh to the World day that
occurs most frequently. This allows us to see if the most frequent day
tends to have more shareshigh.

``` r
## creates plotting object of shares
a <- ggplot(data_channel_train, aes(x=shares))

## histogram of shares 
a+geom_histogram(color= "red", fill="blue")+ ggtitle("Shares histogram")
```

![](World_files/figure-gfm/shares%20histogram-1.png)<!-- -->

Above we can see the frequency distribution of shares of the World data
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

![](World_files/figure-gfm/col%20graph-1.png)<!-- -->

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

    ##              LDA_02           Avg.Words            Rate.Neg         Max.Neg.Pol 
    ##        -0.099373019        -0.062391097        -0.052279145        -0.051859399 
    ##        Rate.Nonstop Rate.Unique.Nonstop                 Wed     Global.Neg.Rate 
    ##        -0.042400308        -0.033902734        -0.032142496        -0.029901810 
    ##            Abs.Subj         Rate.Unique         Min.Pos.Pol         Avg.Pos.Pol 
    ##        -0.028606904        -0.025468439        -0.019813814        -0.019719763 
    ##         Avg.Neg.Pol                Tues                 Fri           n.Content 
    ##        -0.016128863        -0.013208403        -0.012905279        -0.010597616 
    ##         Avg.Min.Key       Min.Worst.Key         Max.Pos.Pol             n.Other 
    ##        -0.004593800        -0.003092323        -0.002569626        -0.000939131 
    ##       Avg.Worst.Key        Min.Best.Key         Min.Neg.Pol            Rate.Pos 
    ##         0.001778954         0.003153378         0.004881212         0.005533450 
    ##       Max.Worst.Key        Avg.Best.Key        Max.Best.Key         Global.Subj 
    ##         0.005848402         0.009761670         0.010863308         0.013252937 
    ##                 Sun                 Mon              LDA_00               Thurs 
    ##         0.014978483         0.015442909         0.015545308         0.019575135 
    ##             Min.Ref                 Sat              LDA_01     Global.Pos.Rate 
    ##         0.021280690         0.021507038         0.022063438         0.024506080 
    ##          Global.Pol               n.Key             Weekend             n.Links 
    ##         0.025503692         0.026187838         0.026645339         0.031125119 
    ##             n.Title         Avg.Max.Key          Title.Subj             Avg.Ref 
    ##         0.033943861         0.035290853         0.035400800         0.036015140 
    ##           Title.Pol             Max.Ref             Abs.Pol            n.Videos 
    ##         0.036234541         0.036998246         0.037160568         0.037890391 
    ##              LDA_04         Avg.Avg.Key            n.Images              LDA_03 
    ##         0.044058564         0.070783591         0.075507471         0.087345018 
    ##              shares 
    ##         1.000000000

``` r
## take the name of the highest correlated variable
highest_cor <-shares_correlations[52]  %>% names()

highest_cor
```

    ## [1] "LDA_03"

``` r
## creats scatter plot looking at shares vs highest correlated variable
g <-ggplot(data_channel_train,  aes(y=shares, x= data_channel_train[[highest_cor]])) 


g+ geom_point(aes(color=as.factor(Weekend))) +geom_smooth(method = lm) + ggtitle(" Highest correlated variable with shares") + labs(x="Highest correlated variable vs shares", color="Weekend")
```

![](World_files/figure-gfm/graph%20of%20shares%20with%20highest%20correlated%20var-1.png)<!-- -->

The above graph looks at the relationship between shares and the
variable with the highest correlation for the World data channel, and
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

![](World_files/figure-gfm/boosted%20tree%20tuning-1.png)<!-- -->

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
                         #rfRMSE=rfRMSE[1],
                          boosted_tree_RMSE =
                           boosted_tree_RMSE[1] )

models_RMSE
```

    ##      linear_1_RMSE linear_2_RMSE boosted_tree_RMSE
    ## RMSE       6099.04       6098.79           6071.12

``` r
## gets the name of the column with the smallest rmse 
smallest_RMSE<-colnames(models_RMSE)[apply(models_RMSE,1,which.min)]

## declares the model with smallest RSME the winner 
paste0(" For ", 
        params$DataChannel, " ", 
       smallest_RMSE, " is the winner")
```

    ## [1] " For World boosted_tree_RMSE is the winner"

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
