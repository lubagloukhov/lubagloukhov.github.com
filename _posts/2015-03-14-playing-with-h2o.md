---
layout: post
category : [play]
tagline: ""
tags : [R, h2o, aws]
---
{% include JB/setup %}


Recently, I joined forces with my classmates Cody and David to partake in the [Kaggle/Microsoft Malware Classification Challenge](https://www.kaggle.com/c/malware-classification). This post details the H2O portion of the efforts but you can read more about the full scope of the  work here: [msan-vs-malware.com](http://msan-vs-malware.com).

Having been introduced to H2O by my colleague [Joe at Revolution Analytics](http://blog.revolutionanalytics.com/2014/04/a-dive-into-h2o.html) and through a couple of meetups at the H2O HQ (most notably, [Find Better Bordeaux with Deep Learning](http://www.meetup.com/Deep-Learning-Applications/events/219471222/)), I was really excited to use this as an opportunity to try it out for myself. 

Two predictive models were trained using H2O running on EC2. Launching of H2O on EC2 requires a [download](http://h2o.ai/download/) of the bleeding edge version and be either modifying and running pre-built scripts from the command line or from the AWS console. As with most things, in practicality, getting up and running was not as straightforward as expected. But that's for another posting.

To connect through R, an install of the bleeding edge H2O R package (shipped within the H2O download mentioned above) was required. Once h2o.init() is run to connect to the running H2O instance, the capabilities of H2O are available in a familiar R syntax. Many of the common R functions such as head(), class(), colnames() and summary() are built out to handle H2O data objects. The close integration with R makes it easy to visually explore aggregations of the data such as the number of files by class in the training corpus shown below.

![class distribution]({{lubagloukhov.github.com}}/assets/Rplot01.jpeg )

We can see the class distribution is unbalanced, with few training files labeled as class 5 — Simda.

~~~ r
library(h2o)
ec2H2O <- h2o.init(ip = ’00.00.00.000’, port =54321) 
pathToData = "https://s3-us-west-2.amazonaws.com/location/filname.txt”
filename.hex = h2o.importFile(ec2H2O, path = pathToData,
							  key = "filename.hex")

classCounts <- as.data.frame(h2o.table(filename.hex["Class"]))
names(classCounts) <- c("class", "count")
ggplot(classCounts) + 
  geom_bar(aes(x=factor(class), y=count, fill=class), stat="identity") + 
  guides(fill=FALSE) + 
  ggtitle("Number of Files by Class") + xlab("Class") + ylab("Count")
~~~

In training the models, the training dataset was split into 80% train and 20% test subsets. Each model was trained on the 80% subset and tested on the other 20%. 

~~~ r
# Train/Test Split
s <- h2o.runif(merged.hex)
merged.train <- merged.hex[s <= 0.8,]
merged.train <- h2o.assign(merged.train, "merged.train")
merged.test <- merged.hex[s > 0.8,]
merged.test <- h2o.assign(merged.test, "merged.test")
~~~ 

h2o.randomForest() trains a random forest model on the 80% subset with 500 trees grown to a maximum depth of 10,000.  h2o.deeplearning() trains a deep learning model on the 80% subset with two hidden layers of 100 neurons and 200 neurons each and 5 epochs. h2o.predict() generates predicted values for the unseen 20% subset based on the trained models. h2o.confusionMatrix() creates a 9x9 matrix of the number of observations falling into each class versus the number of observations predicted as each class.

~~~ r
# Random Forest Training
xs = colnames(filename.hex)[-1] # vector of all feature names
merged.rf = h2o.randomForest(y = "virus", x = xs, 
                             data = merged.train, 
                             ntree = 500, depth = 10000)

# Radom Forest Testing 
merged.rf.pred = h2o.predict(merged.rf, merged.test[xs])
h2o.confusionMatrix(merged.rf.pred[,1], merged.test[“virus"])


# Deep Learning Training
merged.dl = h2o.deeplearning(y = "virus", x = xs, 
                     data = merged.train, 
                     hidden = c(100, 200),
                     epochs = 5)

# Deep Learning Testing
merged.dl.pred = h2o.predict(merged.dl, merged.test[xs])
h2o.confusionMatrix(merged.dl.pred[,1], merged.test[“virus"])
~~~

A visualization of the confusion matrices was constructed using reshape2 and ggplot2. 

~~~r
library(reshape2)
library(ggplot2)
#  Confusion Matrix (repeat for DL & RF)
DLCM <- h2o.confusionMatrix(merged.dl.pred[,1], merged.test["virus"])
actualTotal <- apply(DLCM[c(1:9),c(1:9)], 1, sum)
DLCMmelt <- as.data.frame(melt(DLCM[c(1:9),c(1:9)]))
DLCMmelt$actualCount <- rep(actualTotal)
DLCMmelt$ofactual <- DLCMmelt$value/DLCMmelt$actualCount

ggplot(data=DLCMmelt) +
  geom_tile(aes(x=Actual, y=Predicted, fill=ofactual),
            color="black", size=0.1) +
  labs(x="Actual",y="Predicted") + 
  geom_text(aes(x=Actual,y=Predicted, 
                label=sprintf("%.2f", ofactual)),
            size=3, colour="black") +
  scale_fill_gradient(low="grey",high="red") 
~~~


The confusion matrices on the 20% testing subset for Random Forests (left) and Deep Learning (right) are shown below (numbers indicate portion of actual correctly predicted). 

![random forest confusion matrix]({{lubagloukhov.github.com}}/assets/Rplot_rfcm.jpeg ) ![deep learning confusion matrix]({{lubagloukhov.github.com}}/assets/Rplot_dlcm.jpeg )

We can see that both models do well in correctly predicting the class for all classes except 5 — Simda.


In the end, it wasn’t the look of a confusion matrix but a log-loss calculation that would determine our ranking. The following code computes log-loss given a vector of labels and the output of h2o.predict(), allowing us to determine the log-loss of our 20% labeled predictions.


~~~r
loglossCalc <- function(actual, predicted) {
    # takes as input a vector of actual labels
    #   and an h2o.predict returned object
    # returns a logloss value
  N <- length(actual)
  pred <- as.data.frame(predicted)
  pred$actual <- actual
  pred$ylogp <- apply(dl.pred,1, function(x) 1*log(x[[x[1]+1]]))
  logloss <- -1/N*sum(dl.pred$ylogp)
  return(logloss)  
}

loglossCalc(actual=as.data.frame(merged.test["virus"])[[1]],
            predicted=merged.rf.pred)
~~~

The Random Forest and Deep Learning models trained on H2O lead to a 0.0988557 and 0.1809695 log-loss, respectively.

In summary, H2O’s integration with R made exploratory analysis intuitive and relatively pain free. However, subsequent attempts to analyze larger datasets split up across multiple files were unsuccessful. H2O is extremely sensitive to column headers (special characters, duplicate names). H2O has no support for merging of data aside from the H2O built-out cbind() and rbind(). As such, to combine multiple files, it is required that all of the files have all of the same columns in the same order, a surprisingly challenging undertaking when dealing with 21 files of ~10MB containing thousands of columns. There is also no support for sorting of data. Adding JSON or similar dictionary—key/value support would allow for efficient storage and analysis of files that may be too sparse for columnar representation. H2O is great at fast, scalable machine learning in an environment many data scientists find comfort in. However, being finicky about data inputs and lacking vital data munging capabilities, H2O is far from a standalone tool even in situations that may not call for as much feature engineering. 

Next Steps:

My initial inclination to use H2O was driven by the anticipated need to deal with much larger (specifically, wider) datasets. However, this data has posed a challenge for H2O's  parsing. This is one of the hurdles I'd like to address in future work. Further, there's still a lot of work that can be done with the existing datasets. For one, I'd like to run through some paramater tuning of the existing random forest and deep learning models. I'd also like to see how feature normalization affects the performance of the deep learning model. Finally, I'd like to address the issu of unbalance classes by either bootstrapping for equally represented classes before model fitting or by fitting a boosted algorithm such as h2o's GBM.