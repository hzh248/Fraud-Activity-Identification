# Fraud-Activity-Identification

---
title: "Case_Competition"
author: "MSBA635_Coh_B_Tm_2"
date: "1/25/2022"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```
  
  
### Members of MSBA 635, Cohort B, Team 2  
**Choukry, Kenza**  
**Eisenman, Dana**  
**Kommareddy, Krithik**  
**Nukala, Sriram**  
**Zhang, Hao**  

Winter 2022 Case Competition Submission


## Loading Packages and Setting Working Directory
The hidden code chunk is used to load the required packages:
 - caret
 - skimr
 - tidyverse
 - ROCR
 - rpart & rpart.plot
 - randomForest
 - xgboost  
 - lubridate  

The code block also sets the working directory. **YOU MUST UPDATE THE WORKING DIRECTORY TO YOUR NEED!** 

Finally, the code block increases the memory allocation for R.

```{r packages_setwd, echo = FALSE, warning = FALSE, message = FALSE}
library(caret)
library(skimr)
library(tidyverse)
library(ROCR)
library(rpart)
library(rpart.plot)
library(randomForest)
library(xgboost)
library(lubridate)

setwd("E:/MSBA/20220103 MSBA635 Data Analytics 2/Case Competition 2022")

memory.limit(24000)

```

## Loading Data

```{r load_data, echo=FALSE}

new_all_train <- read.csv(file = "New_Alliance_data_.csv", header=T)

summary(new_all_train)
```

## Step 0 - Exploratory Data Analysis (EDA)  

There are 12,601 observations in the training set and 1,399 in the testing se.t  
``` {r EDA}
# Step 0 - Exploratory Data Analysis (EDA)
summaryStats <- skim(new_all_train)
summaryStats
```

## Step 0a - Feature Engineering  
During EDA, the team determined that additional features would be beneficial for creating a useful model.

Code is mirrored to create the same features in both the training and testing data.

``` {r feature_engineering_train, echo = FALSE}

# Conduct data cleaning and feature engineering
# new_all_train <- read.csv(file = "New_Alliance_data_.csv", header=T)


new_all_train <- new_all_train %>%
  # Creates a "Count" column with all 1s for use in creating plots.
  mutate(Count = 1, 
         # Some TRAN_DT are in Excel's format and must be converted.
         TRAN_DT = if_else(substr(TRAN_DT, 1,2) == "44", 
                           as.Date(as.numeric(TRAN_DT), origin = "1899-12-30"),
                           mdy(TRAN_DT)),
         # Convert CUST_SINCE_DT to a date.
         CUST_SINCE_DT = mdy(CUST_SINCE_DT),
         # Convert ACTVY_DT to a date.
         ACTVY_DT = mdy(ACTVY_DT),
         #Pull transaction time out of TRAN_TS
         TRAN_TS_Time = if_else(substr(TRAN_TS, 1,2) == "44",
                                "12:00",
                               substr(TRAN_TS, nchar(TRAN_TS)-4, 
                                       nchar(TRAN_TS))),
         # Extract the hour
         TRAN_TS_Hour = as.numeric(substr(TRAN_TS_Time, 1, 2)),
         # Create column with the day of the week for the transaction
         TRAN_DayOfWeek = factor(weekdays(TRAN_DT),
                                 levels=c("Sunday",
                                          "Monday",
                                          "Tuesday",
                                          "Wednesday",
                                          "Thursday",
                                          "Friday",
                                          "Saturday")),
         # Column with the number of years a customer has been a member of the bank
         Cust_Since_Years = as.numeric(difftime(mdy("12/31/2021"),
                                                CUST_SINCE_DT,
                                                unit="days")/365.25),
         # Flag if a customer's age is too young to be a customer since...
         Age_Mismatch = if_else(CUST_AGE <= Cust_Since_Years, "Yes", "No"),
         # Bin customer age
         Cust_Age_Bin = factor(case_when(CUST_AGE < 21 ~ "Under21",
                                         CUST_AGE >= 21 & CUST_AGE < 35 ~ "21to35",
                                         CUST_AGE >= 35 & CUST_AGE < 55 ~ "35to55",
                                         CUST_AGE >= 55 ~ "55Over"),
                               levels = c("Under21", "21to35", "35to55", "55Over")),
         # Flag if the customer has been a member over 105 years.
         Cust_Since_Mismatch = if_else(CUST_SINCE_DT >= 105, "Yes", "No"),
         # Compute the number of days between the last time they updated their password
         # and the date of the transaction
         Days_Since_PW_Change = as.numeric(difftime(mdy_hm(TRAN_TS), 
                                         mdy_hm(PWD_UPDT_TS), 
                                         units = "days")),
         # Compute the number of days between the last time they updated their phone number
         # and the date of the transaction
         Days_Since_Ph_Num_Change = as.numeric(difftime(mdy_hm(TRAN_TS), 
                                         mdy_hm(PH_NUM_UPDT_TS), 
                                         units = "days"))
  )

# Some of the response variables are no longer needed.
# ACTN_CD, ACTN_INTNL_TXT, and TRAN_TYPE_CD have only 1 unique value
# PWD_UPDT_TS, PH_NUM_UPDT_TS, and TRAN_TS_Time are causing issues trying to convert. 
# Since we used them to create new variables, we are not losing any information by deleting them.
new_all_train <- select(new_all_train, -c(ACTN_CD, ACTN_INTNL_TXT, 
                                          TRAN_TYPE_CD, TRAN_TS, 
                                          PWD_UPDT_TS, PH_NUM_UPDT_TS,
                                          TRAN_TS_Time))



summaryStats2 <- skim(new_all_train)
summaryStats2
```
``` {r plot_new_features_train, echo = FALSE}

plot_fill_DayOfWeek <- ggplot(subset(new_all_train, !is.na(TRAN_DayOfWeek)), 
                         aes(x = TRAN_DayOfWeek, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "fill", stat = "identity") +
  ggtitle("Percent Distribution of Fraud by Day of the Week") +
  theme(legend.title = element_blank())

plot_stack_DayOfWeek <- ggplot(subset(new_all_train, !is.na(TRAN_DayOfWeek)), 
                         aes(x = TRAN_DayOfWeek, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Day of the Week") +
  theme(legend.title = element_blank())

plot_fill_Cust_Age_Bin <- ggplot(subset(new_all_train, !is.na(Cust_Age_Bin)), 
                         aes(x = Cust_Age_Bin, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "fill", stat = "identity") +
  ggtitle("Percent Distribution of Fraud by Customer Age") +
  theme(legend.title = element_blank())

plot_stack_Cust_Age_Bin <- ggplot(subset(new_all_train, !is.na(Cust_Age_Bin)), 
                         aes(x = Cust_Age_Bin, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Customer Age") +
  theme(legend.title = element_blank())

# plot_fill_TRAN_TS_Time <- ggplot(subset(new_all_train, !is.na(TRAN_TS_Time)), 
#                          aes(x = TRAN_TS_Time, fill = FRAUD_NONFRAUD, y = Count)) +
#   geom_bar(position = "fill", stat = "identity") +
#   ggtitle("Percent Distribution of Fraud by Time of Day") +
#   theme(legend.title = element_blank())
# 
# plot_stack_TRAN_TS_Time <- ggplot(subset(new_all_train, !is.na(TRAN_TS_Time)), 
#                          aes(x = TRAN_TS_Time, fill = FRAUD_NONFRAUD, y = Count)) +
#   geom_bar(position = "stack", stat = "identity") +
#   ggtitle("Distribution of Fraud by Time of Day") +
#   theme(legend.title = element_blank())

plot_fill_TRAN_TS_Hour <- ggplot(subset(new_all_train, !is.na(TRAN_TS_Hour)), 
                         aes(x = TRAN_TS_Hour, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "fill", stat = "identity") +
  ggtitle("Percent Distribution of Fraud by Hour of the Day") +
  theme(legend.title = element_blank())

plot_stack_TRAN_TS_Hour <- ggplot(subset(new_all_train, !is.na(TRAN_TS_Hour)), 
                         aes(x = TRAN_TS_Hour, fill = FRAUD_NONFRAUD, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Hour of the Day \n('0' = Midnight to 12:59 AM)") +
  theme(legend.title = element_blank())

plot_box_Days_Since_PW_Change <- ggplot(subset(new_all_train, 
                                               !is.na(Days_Since_PW_Change)), 
                                        aes(x = Days_Since_PW_Change, 
                                            y = FRAUD_NONFRAUD, 
                                            fill = FRAUD_NONFRAUD)) +
  geom_boxplot() +
  ggtitle("Boxplot Distribution  of Days Since Password Change \n Separated by Fraud") +
  theme(legend.position = "none")




plot_fill_DayOfWeek

plot_stack_DayOfWeek

plot_fill_Cust_Age_Bin  

plot_stack_Cust_Age_Bin  

# plot_fill_TRAN_TS_Time  
# 
# plot_stack_TRAN_TS_Time

plot_fill_TRAN_TS_Hour  

plot_stack_TRAN_TS_Hour

plot_box_Days_Since_PW_Change 
```


``` {r feature_engineering_test, echo = FALSE}

# Conduct data cleaning and feature engineering
new_all_test <- read.csv(file = "New_Alliance_holdout_.csv", header=T)


new_all_test <- new_all_test %>%
  # Creates a "Count" column with all 1s for use in creating plots.
  mutate(Count = 1, 
         # Some TRAN_DT are in Excel's format and must be converted.
         TRAN_DT = if_else(substr(TRAN_DT, 1,2) == "44", 
                           as.Date(as.numeric(TRAN_DT), origin = "1899-12-30"),
                           mdy(TRAN_DT)),
         # Convert CUST_SINCE_DT to a date.
         CUST_SINCE_DT = mdy(CUST_SINCE_DT),
         # Convert ACTVY_DT to a date.
         ACTVY_DT = mdy(ACTVY_DT),
         #Pull transaction time out of TRAN_TS
         TRAN_TS_Time = if_else(substr(TRAN_TS, 1,2) == "44",
                                "12:00",
                               substr(TRAN_TS, nchar(TRAN_TS)-4, 
                                       nchar(TRAN_TS))),
         # Extract the hour
         TRAN_TS_Hour = as.numeric(substr(TRAN_TS_Time, 1, 2)),
         # Create column with the day of the week for the transaction
         TRAN_DayOfWeek = factor(weekdays(TRAN_DT),
                                 levels=c("Sunday",
                                          "Monday",
                                          "Tuesday",
                                          "Wednesday",
                                          "Thursday",
                                          "Friday",
                                          "Saturday")),
         # Column with the number of years a customer has been a member of the bank
         Cust_Since_Years = as.numeric(difftime(mdy("12/31/2021"),
                                                mdy(CUST_SINCE_DT),
                                                unit="days")/365.25),
         # Flag if a customer's age is too young to be a customer since...
         Age_Mismatch = if_else(CUST_AGE <= Cust_Since_Years, "Yes", "No"),
         # Bin customer age
         Cust_Age_Bin = factor(case_when(CUST_AGE < 21 ~ "Under21",
                                         CUST_AGE >= 21 & CUST_AGE < 35 ~ "21to35",
                                         CUST_AGE >= 35 & CUST_AGE < 55 ~ "35to55",
                                         CUST_AGE >= 55 ~ "55Over"),
                               levels = c("Under21", "21to35", "35to55", "55Over")),
         # Flag if the customer has been a member over 105 years.
         Cust_Since_Mismatch = if_else(CUST_SINCE_DT >= 105, "Yes", "No"),
         # Compute the number of days between the last time they updated their password
         # and the date of the transaction
         Days_Since_PW_Change = as.numeric(difftime(mdy_hm(TRAN_TS), 
                                         mdy_hm(PWD_UPDT_TS), 
                                         units = "days")),
         # Compute the number of days between the last time they updated their phone number
         # and the date of the transaction
         Days_Since_Ph_Num_Change = as.numeric(difftime(mdy_hm(TRAN_TS), 
                                         mdy_hm(PH_NUM_UPDT_TS), 
                                         units = "days"))
  )

# Some of the response variables are no longer needed.
# ACTN_CD, ACTN_INTNL_TXT, and TRAN_TYPE_CD have only 1 unique value
# PWD_UPDT_TS, PH_NUM_UPDT_TS, and TRAN_TS_Time are causing issues trying to convert. 
# Since we used them to create new variables, we are not losing any information by deleting them.
new_all_test <- select(new_all_test, -c(ACTN_CD, ACTN_INTNL_TXT, 
                                          TRAN_TYPE_CD, TRAN_TS, 
                                          PWD_UPDT_TS, PH_NUM_UPDT_TS,
                                          TRAN_TS_Time))



summaryStatsTest <- skim(new_all_test)
summaryStatsTest
```


``` {r plot_new_features_test, echo = FALSE}
# plot_fill_DayOfWeek <- ggplot(subset(new_all_test, !is.na(TRAN_DayOfWeek)), 
#                          aes(x = TRAN_DayOfWeek, y = Count)) +
#   geom_bar(position = "fill", stat = "identity") +
#   ggtitle("Percent Distribution of Fraud by Day of the Week\n\n**TEST DATA**") +
#   theme(legend.title = element_blank())

plot_stack_DayOfWeek <- ggplot(subset(new_all_test, !is.na(TRAN_DayOfWeek)), 
                         aes(x = TRAN_DayOfWeek, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Day of the Week\n\n**TEST DATA**") +
  theme(legend.title = element_blank())

# plot_fill_Cust_Age_Bin <- ggplot(subset(new_all_test, !is.na(Cust_Age_Bin)), 
#                          aes(x = Cust_Age_Bin, y = Count)) +
#   geom_bar(position = "fill", stat = "identity") +
#   ggtitle("Percent Distribution of Fraud by Customer Age\n\n**TEST DATA**") +
#   theme(legend.title = element_blank())

plot_stack_Cust_Age_Bin <- ggplot(subset(new_all_test, !is.na(Cust_Age_Bin)), 
                         aes(x = Cust_Age_Bin, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Customer Age\n\n**TEST DATA**") +
  theme(legend.title = element_blank())

# plot_fill_TRAN_TS_Time <- ggplot(subset(new_all_test, !is.na(TRAN_TS_Time)), 
#                          aes(x = TRAN_TS_Time, y = Count)) +
#   geom_bar(position = "fill", stat = "identity") +
#   ggtitle("Percent Distribution of Fraud by Time of Day\n\n**TEST DATA**") +
#   theme(legend.title = element_blank())
# 
# plot_stack_TRAN_TS_Time <- ggplot(subset(new_all_test, !is.na(TRAN_TS_Time)), 
#                          aes(x = TRAN_TS_Time, y = Count)) +
#   geom_bar(position = "stack", stat = "identity") +
#   ggtitle("Distribution of Fraud by Time of Day\n\n**TEST DATA**") +
#   theme(legend.title = element_blank())

# plot_fill_TRAN_TS_Hour <- ggplot(subset(new_all_test, !is.na(TRAN_TS_Hour)), 
#                          aes(x = TRAN_TS_Hour, y = Count)) +
#   geom_bar(position = "fill", stat = "identity") +
#   ggtitle("Percent Distribution of Fraud by Hour of the Day\n\n**TEST DATA**") +
#   theme(legend.title = element_blank())

plot_stack_TRAN_TS_Hour <- ggplot(subset(new_all_test, !is.na(TRAN_TS_Hour)), 
                         aes(x = TRAN_TS_Hour, y = Count)) +
  geom_bar(position = "stack", stat = "identity") +
  ggtitle("Distribution of Fraud by Hour of the Day \n('0' = Midnight to 12:59 AM)\n\n**TEST DATA**") +
  theme(legend.title = element_blank())

plot_box_Days_Since_PW_Change <- ggplot(subset(new_all_test, 
                                               !is.na(Days_Since_PW_Change)), 
                                        aes(x = Days_Since_PW_Change, 
                                            y = Count)) +
  geom_boxplot() +
  ggtitle("Boxplot Distribution  of Days Since Password Change \n Separated by Fraud\n\n**TEST DATA**") +
  theme(legend.position = "none")




# plot_fill_DayOfWeek

plot_stack_DayOfWeek

# plot_fill_Cust_Age_Bin  

plot_stack_Cust_Age_Bin  

# plot_fill_TRAN_TS_Time  
# 
# plot_stack_TRAN_TS_Time

# plot_fill_TRAN_TS_Hour  

plot_stack_TRAN_TS_Hour

plot_box_Days_Since_PW_Change
```


``` {r save_csv, echo = FALSE}
saveName <- str_c("New_All_Data_with_New_Vars_", Sys.Date(), ".csv", sep="")
write_csv(new_all_train, saveName)

saveName2 <- str_c("New_All_Holdout_with_New_Vars_", Sys.Date(), ".csv", sep="")
write_csv(new_all_test, saveName2)
```




```{r create_dummy_variables, echo = FALSE}

# Step 1  - partition data and dummy variables

# Get all predictors without response variable
new_all_train_predictors <- select(new_all_train,-FRAUD_NONFRAUD)

dummies_model <- dummyVars(~., data = new_all_train_predictors)

new_all_predictors_dummy <- data.frame(predict(dummies_model, 
                                               newdata = new_all_train))

new_all_train <- cbind(FRAUD_NONFRAUD = new_all_train$FRAUD_NONFRAUD,
                       new_all_predictors_dummy)

```


## Train the models

The team trained three model types and compared the results to select the best model.
 1) Classification Regression
 2) Random Forest
 3) XGBoost


### Classification Model
``` {r train_classification_model}

# new_all_class_model <- train(FRAUD_NONFRAUD ~ .,
#                              data = new_all_train,
#                              method = "rpart",
#                              trControl = trainControl(method = "cv",
#                                                      number = 5,
#                                                      ## Estimate class probabilities
#                                                      classProbs = TRUE,
#                                                      #needed to get ROC
#                                                      summaryFunction = twoClassSummary),
#                              metric="ROC")
# 
# new_all_class_model
# 
# rpart.plot(new_all_class_model$finalModel, type=5)
# 
# # Get predicted probabilities 
# predprob_credit<-predict(new_all_class_model , new_all_test, type="prob")
# 
# 
# 
# print("Classification Model Successfully Trained")
```


``` {r train_random_forest}


# print("Random Forest Model Successfully Trained")
```



``` {r train_xgboost}


# print("XGBoost Model Successfully Trained")
```
