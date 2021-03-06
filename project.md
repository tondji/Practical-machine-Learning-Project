---
  title: "Practical machine learning Course Project"
date: "September 19, 2017"
output: 
  html_document: 
  keep_md: yes
---
  
  ```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE, message = FALSE, warning = FALSE)
```

```{r}
library(dplyr)
library(caret)
library(randomForest)

urlTraining <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
urlTesting <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
fileTraining <- "training.csv"
fileTesting <- "testing.csv"
if (!file.exists(fileTraining))
  download.file(url = urlTraining, fileTraining)
if (!file.exists(fileTesting))
  download.file(url = urlTesting, fileTesting)
data <- read.csv(fileTraining, na.strings=c("NA", "", "#DIV/0!"))
```
## Foreword
In this course project we take Human Activity Data to try to pedict in wich manner people have done an excercise, how well have they done it.  

## Sampling data
We split our data from training file into two parts: one for training models, another - testing.
```{r echo=TRUE}
set.seed(1377)
inTrain <- createDataPartition(y = data$classe, p = 0.75, list = FALSE)
training <- data[inTrain,]
testing <- data[-inTrain,]
```
Let's see whether our training random sample contain a fair amount of observations from every user with every 'classe' from original data
```{r}
options(digits=3)
table(training$user_name, training$classe) /
table(data$user_name, data$classe)
rm(data)
```

## Cleaning data and some exploratory data analysis
```{r echo=TRUE}
dim(training)
```
Data has `r dim(training)[1]` observations and `r dim(training)[2]` variables.  
But many columns consist of mostly NAs 
```{r}
num_NAs <- unlist(lapply(1:160, function(i) { sum(is.na(training[, i])) } ))
head(num_NAs , 44)
```
These columns are
```{r echo=TRUE}
mostlyNA <- unlist(lapply(1:160, function(i) {if (sum(is.na(training[, i])) >
dim(training)[1] * .95)
i } ))
head(colnames(training)[mostlyNA], 26)
```
In total `r length(mostlyNA)` columns. These variables are explained by a variable new_window, which are 'yes' for the rows that contain not-NA values. With the names min, max, avg, var, stddev etc. it's obvious that these variables contain summarising info for each window. Therefore, we remove these variables and new_window variable itself.  
We also remove the next variables that are not covariants in our model: timestamp variables, X variable is just a row number, user_name and num_window is just a window number.  
```{r echo=TRUE}
training <- training %>%
  select(-mostlyNA, -new_window, -X, -raw_timestamp_part_1, -raw_timestamp_part_2, 
         -cvtd_timestamp, -user_name, -num_window)
```
```{r echo=TRUE}
dim(training)
```
Final data has only 53 variables with the last one as an outcome. Every covariate is a numeric value while outcome is a factor with 5 levels: A,B,C,D,E.  We perform the same cleaning on testing data sample.  
```{r}
testing <- testing %>%
  select(-mostlyNA, -new_window, -X, -raw_timestamp_part_1, -raw_timestamp_part_2, 
         -cvtd_timestamp, -user_name, -num_window)
```

## Training and predicting
We are unable to use "glm" model as the outcome has more than two classes.  
Trees model using "rpart" from caret will look like this:
  ```{r echo=TRUE}
treeFit <- train(classe ~ ., method = "rpart", data = training)
preds <- predict(treeFit, newdata = testing)
(treeTab <- table(testing$classe, preds))
```
Training with trees gives us very little accuracy: `r sum(diag(treeTab)) / sum (treeTab)`  
But it takes too long for more sophisticated methods, like bagging, random forest or boosting to train a model fit. For this issue we create a smaller partition of our traning data to figure out which covariates are the most important ones.
```{r echo=TRUE}
inSmallTrain <- createDataPartition(y = training$classe, p = 0.3, list = FALSE)
smallTraining <- training[inSmallTrain,]
rfSmallFit <- randomForest(smallTraining[,-53], smallTraining$classe)
x <- varImp(rfSmallFit)
x[order(x[,1], decreasing = TRUE),]
mostImp <- head(rownames(x)[order(x[,1], decreasing = TRUE)], 12)
```
We take the fist 12 covariates, that are `r mostImp`.  
And build a model fit using random forest with all training data within selected columns.
```{r echo=TRUE}
impTrain <- training[, c(mostImp, "classe")]
rfFullFit <- randomForest(classe ~ ., data = impTrain)
predTest <- predict(rfFullFit, newdata = testing[, c(mostImp, "classe")])
(tab <- table(testing$classe, predTest))
```
Accuracy: `r sum(diag(tab)) / sum (tab)`  

## Summary
Model built on random forest has high accuracy `r sum(diag(tab)) / sum (tab) * 100`% and the expected out of the sample error is `r 1 - sum(diag(tab)) / sum (tab)`. For reaching thet level of accuracy 12 covarians were used: `r mostImp`. With that number of covariants took not too long, not longer than "rpart" with all covariates included.
