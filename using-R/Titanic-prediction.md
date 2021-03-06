# Tutorial for "Titanic-prediction"

Tutorial created for the kaggle competition: [Titanic: Machine Learning from Disaster](https://www.kaggle.com/c/titanic-gettingStarted)

## Data Processing

Load the training and testing data:
```{r echo = TRUE, cache = TRUE}
training = read.csv("train.csv")
testing = read.csv("test.csv")
dim(training)
dim(testing)
```

```{r echo = TRUE}
str(training)
summary(training)
```

## Visualizations

Randomly pick up 5 variables and create a scatter-plot matrix:
```{r echo = TRUE}
library(caret)
featurePlot(x = training[, c(3, 5, 6, 8, 11)], y = training$Survived, plot = "pairs", auto.key = list(columns = 5))
```

## Pre-Processing

Select useful predictors
```{r echo = TRUE}
trainingSel = subset(training, select = c('Survived', 'Pclass', 'Sex', 'Age', 'SibSp', 'Parch', 'Embarked'))
testingSel = subset(testing, select = c('Pclass', 'Sex', 'Age', 'SibSp', 'Parch', 'Embarked'))
head(trainingSel)
head(testingSel)
```

Remove Embaked with empty value
```{r echo = TRUE}
trainingSel = subset(trainingSel, Embarked != '')
trainingSel$Embarked = droplevels(trainingSel$Embarked, "")
testingSel = subset(testingSel, Embarked != '')
```

Replace NA age with the median of Age 
```{r echo = TRUE}
agemedian = median(trainingSel$Age, na.rm = TRUE)
trainingSel$Age = replace(trainingSel$Age, is.na(trainingSel$Age), agemedian)
testingSel$Age = replace(testing$Age, is.na(testing$Age), agemedian)
```


Separate the predictors and class labels.
```{r echo = TRUE}
trainingVars = trainingSel[, c(2:7)]
objLabels = trainingSel[, 1]
objLabels = as.factor(objLabels)
testingVars = testingSel
testingPsgIds = testing[, 1]
```

## Training and Tuning through Cross Validation

#### Data-Spliting
The pre-processed training data is split into two subsets: 75% for training prediction models and 25% as validation set for evaluating error rates and selecting models.

```{r echo = TRUE}
inTrain = createDataPartition(objLabels, p = 0.75, list = FALSE)
forTrainingVars = trainingVars[inTrain, ]
forTestingVars = trainingVars[-inTrain, ]
forTrainingLabels = objLabels[inTrain]
forTestingLabels = objLabels[-inTrain]
```

#### Estimating a Baseline Error Rate

For this classification problem, I first train a decision tree and evaluate its error rate on the validation set. I will use the error rate of the decision tree as a baseline error rate. In the later part of the document, I will train other models and use cross validation to select the best one. 

Train a decision tree using the caret package.
```{r echo = TRUE}
set.seed(2345)
modeldt = train(y = forTrainingLabels, x = forTrainingVars, method = "rpart")
```

Predict the labels for the validation set and estimate the error rate.
```{r echo = TRUE}
preddt = predict(modeldt, forTestingVars)
confusionMatrix(forTestingLabels, preddt)
```

The accuracy of the decision tree is (0.8198). To improve the accuracy, I will train several different models in the following. 

#### Training Different Models and Estimating Error Rates

##### Will Naive Bayes Model Improve the Accuracy?

Train a naive Bayes model and estimate its accuracy. 

```{r echo = TRUE}
set.seed(2345)
library(klaR)
library(MASS)
```

```{r echo = TRUE, warning = FALSE}
modelnb = train(y = forTrainingLabels, x = forTrainingVars, method = "nb", )
```

```{r echo = TRUE, cache = TRUE, warning = FALSE}
prednb = predict(modelnb, forTestingVars)
confusionMatrix(forTestingLabels, prednb)
```

The accuracy of the naive Bayes model is (0.8108). 

##### Building Random Forest Models

Train a random forest model using the the training data as used in training the naive Bayes model.
```{r echo = TRUE, warning = FALSE}
modelrf = train(y = forTrainingLabels, x = forTrainingVars, method = "rf", )
predrf = predict(modelrf, forTestingVars)
confusionMatrix(forTestingLabels, predrf)
```

The accuracy of the random forest model trained by using all the available training data is (0.8378). 

##### Will Feature Selection Help?
I am curious about whether we can select a smaller set of features/predictors to further improve the training process, e.g., better accuracy or faster training time. 

First,  create an integer vector for the specific subset sizes of the predictors that should be tested.
```{r echo = TRUE}
subsets <- c(1:6)
```

Use recursive feature elimination via caret to find the important features. We train a series random forest models and select features through repeated cross validations on different sizes of feature sets. 

```{r echo = TRUE, warning = FALSE}
set.seed(2345)

library(Hmisc)

ctrl <- rfeControl(functions = rfFuncs,
                   method = "repeatedcv",
                   repeats = 5,
                   verbose = FALSE)

rfProfile <- rfe(forTrainingVars, forTrainingLabels, 
                 sizes = subsets,
                 rfeControl = ctrl)

rfProfile
```

##### OK. Let Us Train a Random Forest Model with only a subset of the Variables

Extract the data.
```{r echo = TRUE}
featureEliTrainingVars = forTrainingVars[, c("Sex", "Pclass", "Age", "Parch", "Embarked")]
featureEliTestingVars = forTestingVars[, c("Sex", "Pclass", "Age", "Parch", "Embarked")]
featureEliFinalTestingVars = testingVars[, c("Sex", "Pclass", "Age", "Parch", "Embarked")]
dim(featureEliTrainingVars)
dim(featureEliTestingVars)
dim(featureEliFinalTestingVars)
```

Train a random forest model using only a subset of the variables.

```{r echo = TRUE, warning = FALSE}
modelrf4vars = train(y = forTrainingLabels, x = featureEliTrainingVars, method = "rf", )
predrf4vars = predict(modelrf4vars, featureEliTestingVars)
confusionMatrix(forTestingLabels, predrf4vars)
```

The accuracy of the model trained by only a subset of the variables is (0.8198). 

## Testing Results

Apply the random forest models trained by the full set of variables and the reduced set of variables to the test set. Check their prediction agreement.

```{r echo = TRUE}
resultsrf4vars = predict(modelrf4vars, featureEliFinalTestingVars)
resultsrf = predict(modelrf, testingVars)
confusionMatrix(resultsrf4vars, resultsrf)
```

The two models agree about (95%) of the test cases. 

Make submission file
```{r echo = TRUE}
resultsrfint = as.numeric(as.character(resultsrf))
ids_results_rf = as.data.frame(cbind(testingPsgIds, resultsrfint))
```





