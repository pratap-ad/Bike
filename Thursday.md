Project\_2
================
Taylor Ashby
10/16/2020

# Introduction

For this analysis, our goal is to predict the daily count of bikes used
from a Washington, D.C.-area bike sharing service. There are several
variables included that we would expect to influence how many bikes are
used. For this analysis, we will focus on the following variables:
season, yr, mnth, workingday, weathersit, atemp, and windspeed.
Descriptions of these variables can be found
[here](https://archive.ics.uci.edu/ml/datasets/Bike+Sharing+Dataset).

We will use two methods to model the *cnt* variable: regression tree and
boosted tree model. A regression tree is a model that splits data into
regions and makes a prediction based on the mean value within a given
region. A boosted tree model is a way to enhance the prediction by
growing trees sequentially by modeling the residuals of the previous
tree.

The first thing we need to is to read in the data. Here we will also
drop the casual and registered variables, as they are classifications
that are not relevant to our analysis of the total daily bike count. We
will then filter to the weekday of interest.

``` r
#Read in bike share data
bike<-read_csv("day.csv")

#Drop casual and registered variables, then filter to desired weekday
bike<-select(bike,-casual,-registered) %>% filter(weekday==params$day)
```

Next we need to split the data into a training and test set.

``` r
#Partition into training and test sets
set.seed(123)
bikeIndex<-createDataPartition(bike$cnt,p=0.7,list=FALSE)
bikeTrain<-bike[bikeIndex,]
bikeTest<-bike[bikeIndex,]
```

Before we begin modeling the *cnt* variable, we should view summary
statistics and plot some of the variables to better understand the data
and their relationships with the variable of interest.

``` r
#View summary statistics for training data
summary(bikeTrain,digits=2)
```

    ##     instant        dteday               season          yr            mnth     
    ##  Min.   : 13   Min.   :2011-01-13   Min.   :1.0   Min.   :0.00   Min.   : 1.0  
    ##  1st Qu.:207   1st Qu.:2011-07-26   1st Qu.:2.0   1st Qu.:0.00   1st Qu.: 4.0  
    ##  Median :346   Median :2011-12-11   Median :3.0   Median :0.00   Median : 7.0  
    ##  Mean   :360   Mean   :2011-12-25   Mean   :2.6   Mean   :0.47   Mean   : 6.7  
    ##  3rd Qu.:519   3rd Qu.:2012-06-01   3rd Qu.:4.0   3rd Qu.:1.00   3rd Qu.: 9.2  
    ##  Max.   :720   Max.   :2012-12-20   Max.   :4.0   Max.   :1.00   Max.   :12.0  
    ##     holiday         weekday    workingday     weathersit       temp          atemp     
    ##  Min.   :0.000   Min.   :4   Min.   :0.00   Min.   :1.0   Min.   :0.16   Min.   :0.15  
    ##  1st Qu.:0.000   1st Qu.:4   1st Qu.:1.00   1st Qu.:1.0   1st Qu.:0.36   1st Qu.:0.37  
    ##  Median :0.000   Median :4   Median :1.00   Median :1.0   Median :0.50   Median :0.49  
    ##  Mean   :0.026   Mean   :4   Mean   :0.97   Mean   :1.4   Mean   :0.51   Mean   :0.48  
    ##  3rd Qu.:0.000   3rd Qu.:4   3rd Qu.:1.00   3rd Qu.:2.0   3rd Qu.:0.66   3rd Qu.:0.62  
    ##  Max.   :1.000   Max.   :4   Max.   :1.00   Max.   :3.0   Max.   :0.81   Max.   :0.83  
    ##       hum         windspeed          cnt      
    ##  Min.   :0.00   Min.   :0.053   Min.   : 623  
    ##  1st Qu.:0.54   1st Qu.:0.136   1st Qu.:3271  
    ##  Median :0.60   Median :0.183   Median :4670  
    ##  Mean   :0.61   Mean   :0.190   Mean   :4627  
    ##  3rd Qu.:0.70   3rd Qu.:0.226   3rd Qu.:6249  
    ##  Max.   :0.94   Max.   :0.442   Max.   :7765

``` r
#view the distribution of the variable of interest
g<-ggplot(bikeTrain,aes(x=cnt,y=..density..))
g+geom_histogram()+labs(title="Histogram of Bike Counts")#how to fix y-axis %s
```

![](Thursday_files/figure-gfm/EDA-1.png)<!-- -->

``` r
#Create scatterplots for variables that are likely to be important predictors
g<-ggplot(bikeTrain,aes(y=cnt,fill=as.factor(workingday)))

g+geom_boxplot()+ylab("Bike Count")+xlab("Is it a Working Day?")+ 
  labs(title="Relationship of workingday to cnt") #why is x-axis -0.2 to 0.2?
```

![](Thursday_files/figure-gfm/EDA-2.png)<!-- -->

``` r
g<-ggplot(bikeTrain,aes(y=cnt,fill=as.factor(weathersit)))

g+geom_boxplot()+ylab("Bike Count")+xlab("What is the weather like")+ 
  labs(title="Relationship of weathersit to cnt") #why is x-axis -0.2 to 0.2?
```

![](Thursday_files/figure-gfm/EDA-3.png)<!-- -->

``` r
g<-ggplot(bikeTrain,aes(y=cnt,fill=as.factor(season)))

g+geom_boxplot()+ylab("Bike Count")+xlab("What season is it?")+ 
  labs(title="Relationship of season to cnt") #why is x-axis -0.2 to 0.2?
```

![](Thursday_files/figure-gfm/EDA-4.png)<!-- -->

It looks like there are a few variables we will not need for modeling.
It will be easier to work with if we simply remove those from the
training data set.

``` r
#first remove variables that are not applicable for prediction of cnt
bikeTrain<-bikeTrain %>% select(-instant,-dteday)

#we can also remove variables where their information is captured by other variables
#i.e., we do not need to include the weekday and workingday indicators since we are
#going to create separate reports for each weekday.
bikeTrain<-bikeTrain %>% select(-holiday,-workingday,-temp, -hum)
```

Now we are ready to fit the models. First we will fit a tree model,
which separates the data into regions and uses the mean of the a given
region to predict the responses. Then we will try a boosted tree model.
This approach grows many trees sequentially, modeling residuals as the
responses. By aggregating over many trees, we lose some
interpretability, but the predictions should be better.

As we fit the models, we are also using “leave one out” cross-validation
to determine the optimal values of the tuning parameters so that we can
get the optimal model fit. The output will automatically use the
bestTune values of these parameters when we go to make predictions.

``` r
#now we are ready to fit the tree
train_control<-trainControl(method="LOOCV")
treeFit<-train(cnt ~ ., data=bikeTrain, method="rpart", 
               preProcess=c("center","scale"),
               trControl=train_control, 
               tuneGrid=NULL)
treeFit$results
treeFit$bestTune

#boosted tree model
boostFit<-train(cnt ~ ., data=bikeTrain, method="gbm", 
                preProcess=c("center","scale"),
                trControl=train_control, 
                tuneGrid=NULL, verbose=FALSE) #why does it produce all those different iterations, with no apparent variation?

boostFit$results
boostFit$bestTune
```

Finally, we will make predictions using our best model fits and data
from the test set. We can compare RMSE to determine which one is best
(smaller RMSE is better).

``` r
#predict values on test set and compare RMSEs
treePred<-predict(treeFit,newdata=dplyr::select(bikeTest,-cnt))
a<- postResample(treePred,bikeTest$cnt)

boostPred<-predict(boostFit,newdata=dplyr::select(bikeTest,-cnt))
b<- postResample(boostPred,bikeTest$cnt)
```

As expected, the boosted tree tends to have lower RMSE when applied to
the test data. By fitting multiple trees sequentially, rather than just
a single tree, the boosting method provides a better prediction.

# Second Part

## Linear model fit

``` r
#Secondary Analysis of the tree fit

Fit2<- lm(cnt ~  season + weathersit + windspeed + atemp, data=bikeTrain)
Fit2
```

    ## 
    ## Call:
    ## lm(formula = cnt ~ season + weathersit + windspeed + atemp, data = bikeTrain)
    ## 
    ## Coefficients:
    ## (Intercept)       season   weathersit    windspeed        atemp  
    ##      3353.9        144.5       -955.8      -4703.5       6480.3

``` r
#predict values on test set and compare RMSEs
Pred2<-predict(Fit2,newdata=dplyr::select(bikeTest,-cnt, season, weathersit, windspeed, atemp))
c<- postResample(Pred2,bikeTest$cnt)
```

\#table the RMSE from both of the model fits

``` r
d<- c(a[1], b[1], c[1])
names(d)<- c("Tree_RMSE","Boosted_RMSE", "LM_RMSE" )
d
```

    ##    Tree_RMSE Boosted_RMSE      LM_RMSE 
    ##     1106.582      544.083     1308.627

From the above table the model having lowest value of RMSE is chosen to
be appropriate to fit the data set.
