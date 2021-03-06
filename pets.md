---
title: "Shelter Animal Outcome"
author: "Yujiao Chen"
date: "April 27, 2016"
output: html_document
---

#Shelter Animal Outcomes
##Help improve outcomes for shelter animals

Every year, approximately 7.6 million companion animals end up in US shelters. Many animals are given up as unwanted by their owners, while others are picked up after getting lost or taken out of cruelty situations. Many of these animals find forever families to take them home, but just as many are not so lucky. 2.7 million dogs and cats are euthanized in the US every year.

![](https://kaggle2.blob.core.windows.net/competitions/kaggle/5039/media/kaggle_pets2.png)  
*Glamour shots of Kaggle's shelter pets are pictured above. From left to right: Shelby, Bailey, Hazel, Daisy, and Yeti.*

Using a dataset of intake information including breed, color, sex, and age from the Austin Animal Center, we're asking Kagglers to predict the outcome for each animal.

We also believe this dataset can help us understand trends in animal outcomes. These insights could help shelters focus their energy on specific animals who need a little extra help finding a new home. 

###Acknowledgements

Kaggle is hosting this competition for the machine learning community to use for data science practice and social good. The dataset is brought to you by [Austin Animal Center](http://www.austintexas.gov/department/animal-services). Shelter animal statistics were taken from the [ASPCA](http://www.aspca.org/animal-homelessness/shelter-intake-and-surrender/pet-statistics).


```{r, results='hide', warning=FALSE, message=FALSE}
library(dplyr)
library(splitstackshape)
library(caret)
library(randomForest)
library(tidyr)
```

####Load in the data. There are 26729 entries with 10 features. 
```{r}
dt <- read.csv("train.csv")

head(dt)
dim(dt)
names(dt)
```

####In this study only dog data are analyzed. Then there are 15595 data entries left. 
```{r}
dt <- dt %>% filter(AnimalType=="Dog")

```
####Let's have a look of the outcomes
```{r}
ggplot(dt, aes(OutcomeType)) + geom_bar()
```

####First we sort out dog's breeds to see if they are mixed.   
There are two cases where a specific dog was considered as a mixed:
1. there is "Mix" in there breed description, e.g.Labrador Retriever Mix
2. there are two breed names for a dog, e.g. Collie Smooth/German Shepherd
the result shows that only about a thousand dogs are pure breeds.
```{r, warning=F, message=F}
dt$Breed <- as.character(dt$Breed)
dt <- cSplit(dt, "Breed", "/")
dt[grep(" Mix", dt$Breed_1),]$Breed_2 <- "Mix"
dt$Breed_1 <- gsub(" Mix", "",dt$Breed_1)
dt <- subset(dt, select=-Breed_3)
dt$Breed_2 <- as.character(dt$Breed_2)
dt$isMix <- ""
dt[is.na(dt$Breed_2)]$isMix <- 0
dt[!is.na(dt$Breed_2)]$isMix <- 1
table(dt$isMix)
dt <- subset(dt, select=-Breed_2)
```


####The next step is find out the popularity of each breed.   
The reason to do this is because there are 1380 different types of breed descriptions in the original data, which is hard for further process. 

I found this website has most popular dog breeds in U.S. http://www.akc.org/news/the-most-popular-dog-breeds-in-america/

I simplified the ranks by assign most popular 1-10 breeds with a popularity of 1; 11-20 with a popularity of 2; and so on. There are top 180-some breeds in the above mentioned list. So the breeds out of this list was assigned a popularity of 19.
```{r}
popularity <- read.csv("popularity.csv")
popularity <- subset(popularity, select=c(BREED, Popularity))
names(popularity) <- c("Breed_1", "popularity") 
popularity <- data.table(popularity)
popularity$Breed_1 <- tolower(popularity$Breed_1)
dt$Breed_1 <- tolower(dt$Breed_1)
popularity$Breed_1 <- gsub("s$", "",popularity$Breed_1)
popularity$Breed_1 <- gsub("s $", "",popularity$Breed_1)
dt$Breed_1 <- gsub("s$","", dt$Breed_1)
```

```{r}
dt <- left_join(dt, popularity)

sum(is.na(dt$popularity))
```

Some mismatch in dog breeds are found in two data set. Some of them are due to abbreviation, some of them are due to simplified or left-out partial phrases. Below are several examples:

-pbgv -> petits bassets griffons vendeens
-chesa bay retr -> chesapeake bay retrievers 
-rhod ridgeback -> rhodesian ridgebacks
-podengo pequeno -> portuguese podengo pequenos 
-flat coat retriever -> flat-coated retrievers
...

So I took the painful process to manually correct them.

```{r}
dt[dt$Breed_1=="alaskan husky"]$popularity <- 6
dt[dt$Breed_1=="american eskimo"]$popularity <- 12
dt[dt$Breed_1=="american bulldog"]$popularity <- 1
dt[dt$Breed_1=="st. bernard rough coat"]$popularity <- 5
dt[dt$Breed_1=="collie smooth"]$popularity <- 4
dt[dt$Breed_1=="dogue de bordeaux"]$popularity <- 7
dt[dt$Breed_1=="podengo pequeno"]$popularity <- 16
dt[dt$Breed_1=="anatol shepherd"]$popularity <- 10
dt[dt$Breed_1=="plott hound"]$popularity <- 15
dt[dt$Breed_1=="chinese sharpei"]$popularity <- 6
dt[dt$Breed_1=="doberman pinsch"]$popularity <- 2
dt[dt$Breed_1=="glen of imaal"]$popularity <- 17
dt[dt$Breed_1=="keeshond"]$popularity <- 9
dt[dt$Breed_1=="rhod ridgeback"]$popularity <- 4
dt[dt$Breed_1=="toy poodle"]$popularity <- 1
dt[dt$Breed_1=="bichon frise"]$popularity <- 5
dt[dt$Breed_1=="chihuahua shorthair"]$popularity <- 3
dt[dt$Breed_1=="german shorthair pointer"]$popularity <- 2
dt[dt$Breed_1=="pit bull"]$popularity <- 8
dt[dt$Breed_1=="american pit bull terrier"]$popularity <- 8
dt[dt$Breed_1=="redbone hound"]$popularity <- 14
dt[dt$Breed_1=="american eskimo"]$popularity <- 12
dt[dt$Breed_1=="chihuahua longhair"]$popularity <- 3
dt[dt$Breed_1=="dachshund wirehair"]$popularity <- 2
dt[dt$Breed_1=="german shepherd"]$popularity <- 1
dt[dt$Breed_1=="miniature poodle"]$popularity <- 1
dt[dt$Breed_1=="wire hair fox terrier"]$popularity <- 10
dt[dt$Breed_1=="american bulldog"]$popularity <- 1
dt[dt$Breed_1=="bedlington terr"]$popularity <- 2
dt[dt$Breed_1=="boykin span"]$popularity <- 11
dt[dt$Breed_1=="chesa bay retr"]$popularity <- 5
dt[dt$Breed_1=="standard poodle"]$popularity <- 1
dt[dt$Breed_1=="siberian husky"]$popularity <- 2
dt[dt$Breed_1=="west highland"]$popularity <- 5
dt[dt$Breed_1=="cavalier span"]$popularity <- 2
dt[dt$Breed_1=="pbgv"]$popularity <- 15
dt[dt$Breed_1=="dachshund longhair"]$popularity <- 2
dt[dt$Breed_1=="staffordshire"]$popularity <- 8
```

```{r}
sum(is.na(dt$popularity))

dt[is.na(dt$popularity)]$popularity <- 19
```


####I also want to add more information about dogs, such as their size.  
I found this website which has information about dog breeds and sizes.
http://www.bil-jac.com/dog-breed-library.php

So I scraped the data from the size, and made it into a .csv file.
This time I simply correct dog breeds in the new "dog size.csv" file, instead of correcting them later as I did for popularity. 

There are four dog sizes: S, M, L, XL
```{r}
size <- read.csv("dog size.csv")
size <- data.table(size)
size$Breed_1 <- tolower(size$Breed_1)
dt <- left_join(dt, size)
dt[is.na(dt$Size)]$Size <- "M"
```


####Next step I convert the AgeuponOutcome from years, months, weeks, days to uniform unit of days.
```{r}
dt <- cSplit(dt, "AgeuponOutcome", " ")

dt$AgeuponOutcome_2 <- as.character(dt$AgeuponOutcome_2)
dt$AgeuponOutcome_3 <- 0
dt[dt$AgeuponOutcome_2 %in% c("year","years")]$AgeuponOutcome_3 <- 365
dt[dt$AgeuponOutcome_2 %in% c("month","months")]$AgeuponOutcome_3 <- 30
dt[dt$AgeuponOutcome_2 %in% c("week","weeks")]$AgeuponOutcome_3 <- 7
dt[dt$AgeuponOutcome_2 %in% c("day","days")]$AgeuponOutcome_3 <- 1

dt$AgeuponOutcome_3 <- as.numeric(dt$AgeuponOutcome_3)
dt <- dt %>% mutate(AgeOutcomeinDays=as.numeric(AgeuponOutcome_1)*AgeuponOutcome_3)

dt <- subset(dt, select = -c(AnimalType,AgeuponOutcome_3,AgeuponOutcome_2,AgeuponOutcome_1))
```

####Then I divided DateTime column into year, month, weekday, and hour.
```{r}
dt <- cSplit(dt, "DateTime", " ")
dt$DateTime_1 <- as.Date(dt$DateTime_1)
dt$year <- year(dt$DateTime_1)
dt$month <- month(dt$DateTime_1)
dt$weekday <- weekdays.Date(dt$DateTime_1)

dt <- cSplit(dt, "DateTime_2", ":")
dt$hour <- dt$DateTime_2_1
dt <- subset(dt, select=-c(DateTime_2_1:DateTime_2_3))
dt <- subset(dt, select=-DateTime_1)

```


####For the dog color    
there is a similar problem as dog breeds - too many unique types. 366 different color descriptions are found in the data set. So I decided to transform the dog color into:
1. Light color
2. Dark color
3. Mixed color

first, the lightColor and darkColor lists are constructed.If a dog has only light color, then he/she is a light color dog; same rule applies for dark color dog. If a dog has both light color and dark color, then he/she is a mixed color dog. If a dog has "Tricolor" in its color description, then also classified as mixed color dog.

```{r}
lightColor <- list("White","Tan","Cream","Fawn", "Buff","Yellow","Gold")
darkColor <- list("Black","Brown","Blue","Chocolate","Liver","Gray","Red","Sable")
dt$colorType <- ""

hasLightColor <- sapply(dt$Color, function(x) {
    sum(sapply(lightColor, function(y){
        sum(grepl(y,x))
    }))
})

hasDarkColor <- sapply(dt$Color, function(x) {
    sum(sapply(darkColor, function(y){
        sum(grepl(y,x))
    }))
})

for(i in 1:length(dt$Color)) {
    if(dt$Color[i]=="Tricolor") {
        dt$colorType[i]<- "Mixed"
    } else if(hasLightColor[i]*hasDarkColor[i]>0){
        dt$colorType[i]<-"Mixed"
    } else if(hasDarkColor[i]>0 & hasLightColor[i]==0) {
        dt$colorType[i] <- "Dark"
    } else if(hasDarkColor[i]==0 & hasLightColor[i]>0) {
        dt$colorType[i] <- "Light"
    }
}
```


Some minor adjustment of the data.
####Also, the dog names are hard for analysis. So I convert this information into hasName, which is a boolean value. 

```{r}
dt$isMix <- as.factor(dt$isMix)

dt$hasName <- ifelse(dt$Name=="", F, T)
table(dt$hasName)

names(dt)
dt <- subset(dt, select=c("AnimalID","SexuponOutcome","isMix","Size","popularity","AgeOutcomeinDays","year","month","weekday","hour","colorType","hasName","OutcomeType"))
dt$month<- as.factor(dt$month)
dt$weekday <- as.factor(dt$weekday)
dt$hour <- as.numeric(dt$hour)
dt$colorType <- as.factor(dt$colorType)
```

```{r,  echo=F,results='hide'}
dt <- dt %>% filter(AnimalID!="A721076")
dt <- dt[!(dt$SexuponOutcome==""),]
```


####Sex vs. Outcome  
It is easy to imagine that dogs get neutered or spayed before they are ready to be adopted (it is a common policy in many animal shelters). The dogs with unknown gender have nearly zero adoption rate, which makes sense.
```{r}
ggplot(dt, aes(x=SexuponOutcome, fill=OutcomeType)) +
    geom_bar(stat="count", position="fill", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Size vs. Outcome  
Show a little variance here. 
```{r}
ggplot(dt, aes(x=Size, fill=OutcomeType)) +
    geom_bar(stat="count", position="fill", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Popularity vs. Outcome  
Seems like the popularity doesn't give us the information we would expected. But seems like that there are still some preference over certain breeds. 
```{r}
ggplot(dt, aes(x=popularity, fill=OutcomeType)) +
    geom_bar(stat="count", position="fill", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Age vs. Outcome  
Age is a dominant factor for outcome. Puppies are likely to be adopted and older dogs are likely to be returned to the owner. 
```{r}
ggplot(dt, aes(x=round(AgeOutcomeinDays/365), fill=OutcomeType)) +
    geom_bar(stat="count", position="fill", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####hasName vs. Outcome  
It is within our expectation that dogs with name are more likely to be returned to the owner. Also the dogs with name show less rate of died and euthanasia. 
```{r}
ggplot(dt, aes(x=hasName, fill=OutcomeType)) +
    geom_bar(stat="count", position="fill", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Month vs. Outcome  
Adoption rate is higher in the winter. Probably it is too hot and humid in the summer for people to pay a visit to the shelter
```{r}
ggplot(dt, aes(x=month, fill=OutcomeType)) +
    geom_bar(stat="count", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Weekday vs. Outcome  
More adoptions occur on weekends
```{r}
ggplot(dt, aes(x=weekday, fill=OutcomeType)) +
    geom_bar(stat="count", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```
 
####Time vs. Outcome  
There is a pattern observed in outcomes by hour of the day. In the afternoon the adoption rate is much higher.
```{r}
ggplot(dt, aes(x=hour, fill=OutcomeType)) +
    geom_bar(stat="count", alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Color vs. Outcome  
No significant difference of outcomes by color groups
```{r}
ggplot(dt[dt$colorType!="",], aes(x=colorType, fill=OutcomeType)) +
    geom_bar(stat="count", position="fill",alpha=0.6) +
    ylab("Relative count by outcome") +
    theme(axis.text.x=element_text(angle=90), legend.position="bottom")
```

####Several model were used for prediction  
1. knn (n=9) Accuracy = 0.49  
2. nnet (size=5, decay=1e-01) Accuracy=0.51  
3. nb Accuracy=0.48  
4. gbm (interaction.depth=3, n.trees=100) Accuracy=0.60  
5. lda Accuracy=0.57  
6. rf Accuracy= 0.61  

```{r, message=FALSE, warning=FALSE}
dt <- subset(dt, select=-c(AnimalID))

inTrain <- createDataPartition(dt$OutcomeType, p=0.8)[[1]]
training <- dt[inTrain,]
testing <- dt[-inTrain,]

knn_mod <- train(OutcomeType ~., method="knn", data=training)

nnet_mod <- train(OutcomeType ~., method="nnet", data=training)

nb_mod <- train(OutcomeType ~., method="nb", data=training)

gbm_mod <- train(OutcomeType ~., method="gbm", data=training)

lda_mod <- train(OutcomeType ~., method="lda", data=training)

rf_mod <- randomForest(OutcomeType ~ ., data = training, ntree = 500, importance = TRUE)

knn_mod
nnet_mod
nb_mod
gbm_mod
lda_mod
rf_mod

```

####The error of each outcome group as well as the most important predicting factors.  
We have fairly good predictions on adoption, while very bad predictions on died and euthanasia. It perhaps due to the small sample of died and euthanasia cases, also due to the nature that death is hard to predict I guess? because the health condition is not a provided information. 
```{r}
plot(rf_mod, ylim=c(0,1))
legend('topright', colnames(rf_mod$err.rate), col=1:6, fill=1:6)

importance    <- importance(rf_mod)
varImportance <- data.frame(Variables = row.names(importance), 
                            Importance = round(importance[ ,'MeanDecreaseGini'],2))
rankImportance <- varImportance %>%
  mutate(Rank = paste0('#',dense_rank(desc(Importance))))
ggplot(rankImportance, aes(x = reorder(Variables, Importance), y = Importance)) +
  geom_bar(stat='identity', fill = 'navy') +
  geom_text(aes(x = Variables, y = 0.5, label = Rank),
    hjust=0, vjust=0.55, size = 4, colour = 'white',
    fontface = 'bold') +
  labs(x = 'Variables', title = 'Relative Variable Importance') +
  coord_flip()
```

####Seems like random forest model has the highest prediction accuracy.  
The evaluation criteria declared by Kaggle is multi-class logarithmic loss. So in addition, I am evaluating the prediction accordingly. My final multi-class loss is 0.93. The No.1 on kaggle has 0.45. My current rank will be 248/411.
```{r}

predicted <- predict(rf_mod, testing, type="vote")

actual <- testing$OutcomeType

Actual <- spread((data.frame(cbind(1:length(actual),actual,rep(1,length(actual))))),actual, V3)
Actual[is.na(Actual)] <- 0
combine <- (cbind(Actual, predicted))

eps = 1e-15
combine$Adoption = pmax(pmin(combine$Adoption, 1-eps), eps)
combine$Died = pmax(pmin(combine$Died, 1-eps), eps)
combine$Euthanasia = pmax(pmin(combine$Euthanasia, 1-eps), eps)
combine$Return_to_owner = pmax(pmin(combine$Return_to_owner, 1-eps), eps)
combine$Transfer = pmax(pmin(combine$Transfer, 1-eps), eps)


sum(combine$`1`*log(combine$Adoption)+combine$`2`*log(combine$Died)+combine$`3`*log(combine$Euthanasia)+combine$`4`*log(combine$Return_to_owner)+combine$`5`*log(combine$Transfer))/-length(actual)
```

##Conclusion
Seems like the most important factor for adoption animal outcome is their age. Puppies are likely to be adopted, and old dogs are likely to be returned to owner. 
