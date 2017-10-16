---
title: 'STAT 420: Data Analysis Project'
author: "Apurva V. Hari, Alok K. Shukla"
date: "11/13/2016"
layout: default
---


```{r setup, echo = FALSE, message = FALSE, warning = FALSE}
options(scipen = 1, digits = 4, width = 80)
library(knitr)
opts_chunk$set(tidy.opts=list(width.cutoff=60),tidy=TRUE)
```

## Team

- Size : 2
- Details :

Name           | NetID
-------------- | -------------
Apurva V. Hari | vhari2
Alok K. Shukla | alokks2





## Introduction

**Who's Your Daddy? Is He Rich Like Me?**
<br/>
\newline
 *A survey of economic mobility across generations in contemporary USA.*
 
The data come from a large study, based on tax records, which allowed researchers to link the income of adults to the income of their parents several decades previously. For privacy reasons, we don't have that individual-level data, but we do have aggregate statistics about economic mobility for several hundred communities, containing most of the American population, and covariate information about those communities.

#### Dataset 

A snippet. (Only first few columns).
```{r kable,message=FALSE,echo=FALSE,warning=FALSE}
# Libraries, Helpers
library(readr)
library(faraway)
library(MASS)
library(ggplot2)
library(ggmap)
library(rpart)
library(rpart.plot)
library(leaps)
library(tree)
library(ggplot2)
library(corrplot)
library(lmtest)
calc_loocv_rmse = function(model) {
sqrt(mean((resid(model) / (1 - hatvalues(model))) ^ 2))
}


plot_fitted_resid = function(model, pointcol = "dodgerblue", linecol = "darkorange") {
  plot(fitted(model), resid(model), 
       col = pointcol, pch = 20, cex = 1.5,
       xlab = "Fitted", ylab = "Residuals")
  abline(h = 0, col = linecol, lwd = 2)
}

plot_qq = function(model, pointcol = "dodgerblue", linecol = "darkorange") {
  qqnorm(resid(model), col = pointcol, pch = 20, cex = 1.5)
  qqline(resid(model), col = linecol, lwd = 2)
}

# Data
mobility <- read.csv("mobility.csv")
mobility$Urban = as.factor(mobility$Urban)
mobilityData = mobility[complete.cases(mobility),]
attach(mobilityData)
knitr::kable(head(mobility)[,1:7])
```

#### Description

The data file `mobility.csv` has information on 741 communities. The variable we want to predict is economic mobility; the rest are predictor variables or covariates.

1. Mobility: The probability that a child born in 1980???1982 into the lowest quin- tile (20%) of household income will be in the top quintile at age 30. Individuals are assigned to the community they grew up in, not the one they were in as adults.
2. Population in 2000.
3. Is the community primarily urban or rural?
4. Black: percentage of individuals who marked black (and nothing else) on cen- sus forms.
5. Racial segregation: a measure of residential segregation by race.
6. Income segregation: Similarly but for income.
7. Segregation of poverty: Specifically a measure of residential segregation for those in the bottom quarter of the national income distribution.
8. Segregation of affluence: Residential segregation for those in the top qarter.
9. Commute: Fraction of workers with a commute of less than 15 minutes.
10. Mean income: Average income per capita in 2000.
11. Gini: A measure of income inequality, which would be 0 if all incomes were perfectly equal, and tends towards 100 as all the income is concentrated among the richest individuals (see Wikipedia, s.v. ???Gini coefficient???).
12. Share 1%: Share of the total income of a community going to its richest 1%.
13. Gini bottom 99%: Gini coefficient among the lower 99% of that community.
14. Fraction middle class: Fraction of parents whose income is between the na- tional 25th and 75th percentiles.
15. Local tax rate: Fraction of all income going to local taxes.
16. Local government spending: per capita.
17. Progressivity: Measure of how much state income tax rates increase with in- come.
18. EITC: Measure of how much the state contributed to the Earned Income Tax Credit (a sort of negative income tax for very low-paid wage earners).
19. School expenditures: Average spending per pupil in public schools.
20. Student/teacher ratio: Number of students in public schools divided by num- ber of teachers.
21. Test scores: Residuals from a linear regression of mean math and English test scores on household income per capita.
22. Highschooldropoutrate:Also,residualsfromalinearregressionofthedropout rate on per-capita income.
23. Colleges per capita
24. College tuition: in-state, for full-time students
25. College graduation rate: Again, residuals from a linear regression of the actual graduation rate on household income per capita.
26. Labor force participation: Fraction of adults in the workforce.
27. Manufacturing: Fraction of workers in manufacturing.
28. Chinese imports: Growth rate in imports from China per worker between 1990 and 2000.
29. Teenage labor: fraction of those age 14???16 who were in the labor force.
30. Migration in: Migration into the community from elsewhere, as a fraction of 2000 population.
31. Migration out: Ditto for migration into other communities.
32. Foreign: fraction of residents born outside the US.
33. Social capital: Index combining voter turnout, participation in the census, and participation in community organizations.
34. Religious: Share of the population claiming to belong to an organized religious body.
35. Violent crime: Arrests per person per year for violent crimes.
36. Singlemotherhood:Numberofsinglefemalehouseholdswithchildrendivided by the total number of households with children.
37. Divorced: Fraction of adults who are divorced.
38. Married: Ditto.
39. Longitude: Geographic coordinate for the center of the community
40. Latitude: Ditto
41. ID: A numerical code, identifying the community.
42. Name: the name of principal city or town.
43. State: the state of the principal city or town of the community.


## Methods
### Exploratory Data Analysis

#### Map of mobility.

```{r,warning=FALSE,message=FALSE,tidy=TRUE}
## Central co-ordinates of the region we are interested in.
usa_center = c(mean(mobility$Longitude),mean(mobility$Latitude))
## Get map
USAMap = ggmap(get_googlemap(center=usa_center, scale=1, zoom=2), extent="normal")
## Plot Mobility
USAMap + geom_point(aes(x=mobility$Longitude, y=mobility$Latitude), data=mobility, col="red", alpha=0.4,size=mobility$Mobility*8) + xlim(range(mobility$Longitude)) + ylim(range(mobility$Latitude))
```

Some visual insights: Dakota, Minnesota, Illinois (yayy!) in the central USA have a higher Mobility than Coastal States. East coast has more data points; west is relatively sparse. 
#### Scatterplots

*Population*

```{r,warning=FALSE,message=FALSE}
  
  popData = as.data.frame(cbind(mobility$Population,mobility$Mobility))
  popData = popData[complete.cases(popData),]
  colnames(popData) = c("Pop","Mobility")
  pred.Pop <- predict(lm(Mobility ~ Pop, data = popData))

  
  p1 <- ggplot(popData, aes(x = Pop, y = Mobility))
 
  p1 + geom_point(col="blue") + geom_line(aes(y = pred.Pop,col="orange"))+ geom_smooth() 
```
<br/>
There's relatively much higher variance in Mobility of regions with less population and they are also regions of highest Mobility.

*Mean household income per capita*

```{r,warning=FALSE,message=FALSE}
  
  incomeData = as.data.frame(cbind(mobility$Income,mobility$Mobility))
  incomeData = incomeData[complete.cases(incomeData),]
  colnames(incomeData) = c("Income","Mobility")
  pred.Inc <- predict(lm(Mobility ~ Income, data = incomeData))

  
  p1 <- ggplot(incomeData, aes(x = Income, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Inc,col="orange"))+geom_smooth()
```
<br/>
Not a clear relationship; middle income groups have highest range of Mobility.

*Racial segregation*

```{r,warning=FALSE,message=FALSE}
  
  raceData = as.data.frame(cbind(mobility$Seg_racial,mobility$Mobility))
  raceData = raceData[complete.cases(raceData),]
  colnames(raceData) = c("Racial_Seg","Mobility")
  pred.Race <- predict(lm(Mobility ~ Racial_Seg, data = raceData))

  
  p1 <- ggplot(raceData, aes(x = Racial_Seg, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Race,col="orange"))+geom_smooth()
```
<br/>
Areas with less racial segregation see higher range of Mobility.

*Income share of the top 1%*

```{r,warning=FALSE,message=FALSE}
  
  incomeData = as.data.frame(cbind(mobility$Share01,mobility$Mobility))
  incomeData = incomeData[complete.cases(incomeData),]
  colnames(incomeData) = c("Share01","Mobility")
  pred.Inc <- predict(lm(Mobility ~ Share01, data = incomeData))

  
  p1 <- ggplot(incomeData, aes(x = Share01, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Inc,col="orange"))+geom_smooth()
```

<br/>
An interesting relationship. Areas with less share in top 1% see a higher range of Mobility.

*Mean school expenditures per pupil*

```{r,warning=FALSE,message=FALSE}
  
  schoolData = as.data.frame(cbind(mobility$School_spending,mobility$Mobility))
  schoolData = schoolData[complete.cases(schoolData),]
  colnames(schoolData) = c("SchoolSpend","Mobility")
  pred.Sch <- predict(lm(Mobility ~ SchoolSpend, data = schoolData))

  
  p1 <- ggplot(schoolData, aes(x = SchoolSpend, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Sch,col="orange"))+geom_smooth()
```
<br/>
More expenditure, more Mobility, intuitive.

*Violent crime rate*

```{r,warning=FALSE,message=FALSE}
  
  crimeData = as.data.frame(cbind(mobility$Violent_crime,mobility$Mobility))
  crimeData = crimeData[complete.cases(crimeData),]
  colnames(crimeData) = c("ViolentCrime","Mobility")
  pred.Sch <- predict(lm(Mobility ~ ViolentCrime, data = crimeData))

  
  p1 <- ggplot(crimeData, aes(x = ViolentCrime, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Sch,col="orange"))+geom_smooth()
```
<br/>

Less crime, more Mobility; intuitive.

*Fraction of workers with short commutes.*

```{r,warning=FALSE,message=FALSE}
  
  commuteData = as.data.frame(cbind(mobility$Commute,mobility$Mobility))
  commuteData = commuteData[complete.cases(commuteData),]
  colnames(commuteData) = c("ShortCommute","Mobility")
  pred.Comm <- predict(lm(Mobility ~ ShortCommute, data = commuteData))

  
  p1 <- ggplot(commuteData, aes(x = ShortCommute, y = Mobility))
 
  p1 + geom_point(col="blue") +
    geom_line(aes(y = pred.Comm,col="orange"))+geom_smooth()
```

<br/>

Nearby jobs, more Mobility :)

**Note:** All of these individual predictors aren't considered in isolation; so the observed variations as they appear on the plots might not really be because of the predictor considered.

### Model Selection

0. Tree model, Correlations

```{r,warning=FALSE,message=FALSE}

  # Dropping unique IDs and Names; 
  # and also States (too many levels.)
  
  drops <- c("Name", "ID","State")
  dataset = mobilityData[ , !(names(mobilityData) %in% drops)]
  
  # Lets see interactions in a tree model
  form <- as.formula(Mobility ~ .)
  model <- rpart(form,data=dataset)
  prp(model)
  
  # Correlations
  data = na.omit(mobility)
  #round(cor(data[sapply(data,is.numeric)], use="pairwise.complete.obs"),2)
  corrplot(cor(data[sapply(data,is.numeric)]),method ="ellipse",
         title =" Correlation Matrix Graph",tl.cex = .5,tl.pos ="lt",tl.col ="dodgerblue" )
  
```  

Most important  explanatory variable is `Commute`; and the threshold value seperating low and high values of `Commute` is $0.511$ . The fact that both limbs are branched means that other variables explain a significant amount of the variation in `Mobility` levels for values of `Commute`. 

High levels of correlation amongst multiple predictors is also confirmed by the `corrplot`.

1. First attempt with linear and quadratic relationships.

```{r,warning=FALSE,message=FALSE} 
  
  # 1st model
  model1 = lm(Mobility~Population+Black+Urban+Seg_racial+Seg_income+Seg_poverty+Seg_affluence+Commute+Income+Gini+Share01+Gini_99+Middle_class+Local_tax_rate+Local_gov_spending+Progressivity+EITC+School_spending+Student_teacher_ratio+Test_scores+HS_dropout+Colleges+Tuition+Graduation+Labor_force_participation+Manufacturing+Chinese_imports+Teenage_labor+Migration_in+Migration_out+Foreign_born+Social_capital+Religious+Violent_crime+Single_mothers+Divorced+Married+Longitude+Latitude+I(Population^2)+I(Black^2)+I(Seg_racial^2)+I(Seg_income^2)+I(Seg_poverty^2)+I(Seg_affluence^2)+I(Commute^2)+I(Income^2)+I(Gini^2)+I(Share01^2)+I(Gini_99^2)+I(Middle_class^2)+I(Local_tax_rate^2)+I(Local_gov_spending^2)+I(Progressivity^2)+I(EITC^2)+I(School_spending^2)+I(Student_teacher_ratio^2)+I(Test_scores^2)+I(HS_dropout^2)+I(Colleges^2)+I(Tuition^2)+I(Graduation^2)+I(Labor_force_participation^2)+I(Manufacturing^2)+I(Chinese_imports^2)+I(Teenage_labor^2)+I(Migration_in^2)+I(Migration_out^2)+I(Foreign_born^2)+I(Social_capital^2)+I(Religious^2)+I(Violent_crime^2)+I(Single_mothers^2)+I(Divorced^2)+I(Married^2)+I(Longitude^2)+I(Latitude^2),data=dataset)
  
  # Model 2
  model2 <- step(model1,trace=0)
  #summary(model2)
  # Insignificant
  summary(model2)$coefficients[summary(model2)$coefficients[ ,4] >= 0.05,]
  
  # Model 3, Removed insignificant from previous
  model3 <- update(model2,~.-I(Seg_poverty^2)-Teenage_labor-Seg_racial-I(EITC^2)-I(Migration_in^2))
  #summary(model3)
  
  # Lets tag this model for compariosn later on
  firstModel = model3
  
```

2. All 2-Way interactions compared. We have $700+$ 2-Way interactions; so we fit them randomly shuffled and distributed in 7 models; since we should only estimate around $100$ predictors in one model; for given dataset size ($~400$).

```{r,warning=FALSE,message=FALSE} 

## 7 models now
model4 = lm(Mobility ~ .+School_spending:Chinese_imports+Chinese_imports:Foreign_born+Middle_class:Teenage_labor+Commute:Migration_in+Labor_force_participation:Single_mothers+Gini:Middle_class+Test_scores:Migration_in+Seg_racial:Graduation+Population:Longitude+Commute:Graduation+Local_tax_rate:HS_dropout+Gini_99:Violent_crime+Migration_in:Married+Colleges:Migration_in+Student_teacher_ratio:Migration_out+Manufacturing:Latitude+Black:Migration_in+Seg_affluence:Graduation+Black:Test_scores+Local_gov_spending:Colleges+Gini:Married+Black:Tuition+Seg_affluence:Progressivity+Black:Middle_class+EITC:Chinese_imports+Seg_affluence:Migration_in+Graduation:Longitude+Test_scores:Social_capital+Seg_poverty:Graduation+Colleges:Single_mothers+Chinese_imports:Religious+Income:Share01+Population:Test_scores+Seg_affluence:Tuition+Local_gov_spending:Longitude+Income:Married+Black:Social_capital+School_spending:Divorced+EITC:Violent_crime+Progressivity:Chinese_imports+Seg_racial:Student_teacher_ratio+Violent_crime:Married+Commute:Income+Migration_out:Violent_crime+Seg_affluence:Income+Colleges:Manufacturing+Seg_income:HS_dropout+Test_scores:Married+Colleges:Divorced+Seg_poverty:Gini_99+Share01:Latitude+Seg_racial:Chinese_imports+Income:Foreign_born+Middle_class:School_spending+Labor_force_participation:Longitude+Progressivity:Violent_crime+Migration_in:Violent_crime+Seg_racial:Migration_in+Black:Share01+Population:Social_capital+Seg_poverty:EITC+Black:Student_teacher_ratio+Student_teacher_ratio:Longitude+School_spending:Religious+Labor_force_participation:Chinese_imports+School_spending:Longitude+Gini:Share01+Migration_out:Religious+Local_tax_rate:Teenage_labor+Seg_affluence:Migration_out+Income:Latitude+Manufacturing:Single_mothers+Local_gov_spending:School_spending+School_spending:Violent_crime+Seg_poverty:Religious+Gini_99:Tuition+Student_teacher_ratio:HS_dropout+Seg_racial:Manufacturing+Manufacturing:Violent_crime+Seg_affluence:Single_mothers+Seg_racial:Local_gov_spending+Gini_99:Migration_in+EITC:Graduation+Population:Foreign_born+Gini_99:Local_gov_spending+EITC:Test_scores+Gini_99:Single_mothers+Seg_racial:Seg_affluence+Population:Income+Local_tax_rate:Graduation+Social_capital:Violent_crime+Seg_affluence:Married+Seg_poverty:Middle_class+Commute:Local_tax_rate+HS_dropout:Latitude+Local_gov_spending:Manufacturing+Seg_poverty:Single_mothers+Population:Single_mothers+Population:Latitude+Migration_in:Divorced,data=dataset)

model5 = lm(Mobility~.+Share01:Violent_crime+Black:EITC+Colleges:Latitude+Tuition:Migration_out+Social_capital:Married+Chinese_imports:Longitude+Seg_affluence:Chinese_imports+Teenage_labor:Migration_out+Gini:Divorced+Population:Tuition+HS_dropout:Graduation+Chinese_imports:Divorced+Gini:Test_scores+Colleges:Social_capital+Labor_force_participation:Divorced+Social_capital:Religious+Seg_income:Commute+Seg_affluence:Test_scores+Manufacturing:Social_capital+School_spending:Latitude+Student_teacher_ratio:Single_mothers+Black:Gini+Seg_poverty:Violent_crime+Foreign_born:Longitude+Local_gov_spending:Married+Progressivity:Single_mothers+Manufacturing:Chinese_imports+Black:Commute+Black:Gini_99+EITC:Manufacturing+Progressivity:Labor_force_participation+Migration_out:Latitude+Seg_affluence:Commute+Commute:Manufacturing+Seg_income:Migration_out+EITC:Foreign_born+Migration_in:Single_mothers+Foreign_born:Married+School_spending:Married+Test_scores:Manufacturing+Test_scores:Graduation+Migration_out:Social_capital+Local_gov_spending:Divorced+Local_gov_spending:Graduation+Seg_poverty:Labor_force_participation+Test_scores:Single_mothers+Commute:Tuition+Local_gov_spending:Migration_in+Income:Local_gov_spending+Gini_99:Progressivity+Population:Seg_affluence+Test_scores:Violent_crime+Black:Teenage_labor+Seg_racial:Labor_force_participation+Progressivity:Student_teacher_ratio+Seg_poverty:HS_dropout+Gini_99:Labor_force_participation+Tuition:Divorced+Local_tax_rate:Colleges+EITC:HS_dropout+Gini:Student_teacher_ratio+Local_gov_spending:Violent_crime+Colleges:Graduation+School_spending:Tuition+Local_gov_spending:Student_teacher_ratio+Black:Single_mothers+Teenage_labor:Violent_crime+Migration_out:Foreign_born+Seg_income:Violent_crime+Share01:Teenage_labor+Black:Labor_force_participation+Student_teacher_ratio:Violent_crime+Colleges:Migration_out+Income:EITC+Seg_affluence:Local_tax_rate+Share01:Social_capital+Seg_racial:Latitude+Colleges:Foreign_born+Gini:Chinese_imports+Gini_99:Student_teacher_ratio+Seg_racial:HS_dropout+Gini:Local_gov_spending+Commute:Single_mothers+Seg_affluence:Manufacturing+Seg_affluence:Social_capital+Seg_racial:Foreign_born+Violent_crime:Single_mothers+Seg_racial:Tuition+Progressivity:Longitude+Seg_affluence:Foreign_born+Labor_force_participation:Violent_crime+Manufacturing:Divorced+Progressivity:Religious+Share01:EITC+Local_tax_rate:Violent_crime+Middle_class:HS_dropout+Local_tax_rate:Divorced+Local_gov_spending:Tuition+Seg_income:Share01+Tuition:Latitude,data=dataset)

model6 = lm(Mobility~.+Commute:Chinese_imports+Migration_out:Longitude+Seg_racial:EITC+Share01:Labor_force_participation+Population:Manufacturing+Commute:Progressivity+Income:Labor_force_participation+Progressivity:Tuition+Local_tax_rate:Migration_out+Seg_racial:Progressivity+Middle_class:Student_teacher_ratio+EITC:Student_teacher_ratio+Income:Religious+Local_gov_spending:HS_dropout+School_spending:Foreign_born+Graduation:Chinese_imports+Seg_racial:Gini+Gini:Foreign_born+Seg_racial:Share01+Seg_poverty:Migration_in+Student_teacher_ratio:Religious+Seg_income:Gini+HS_dropout:Colleges+Progressivity:Married+Share01:Foreign_born+Seg_income:Labor_force_participation+Middle_class:Manufacturing+HS_dropout:Divorced+Labor_force_participation:Social_capital+Test_scores:Religious+Income:Chinese_imports+Seg_income:Middle_class+Graduation:Labor_force_participation+Migration_in:Migration_out+Gini:Gini_99+Population:Local_tax_rate+EITC:School_spending+Seg_poverty:Student_teacher_ratio+Income:Gini+EITC:Religious+Local_tax_rate:Longitude+Local_tax_rate:Religious+HS_dropout:Tuition+Student_teacher_ratio:Test_scores+Gini:Longitude+Share01:Religious+Middle_class:Chinese_imports+Tuition:Violent_crime+Seg_poverty:Foreign_born+Seg_poverty:Migration_out+Foreign_born:Latitude+Population:Labor_force_participation+Test_scores:Chinese_imports+Gini:Migration_out+Gini_99:Foreign_born+Gini:Labor_force_participation+Income:Test_scores+Middle_class:Tuition+Seg_income:Progressivity+Gini:Migration_in+Graduation:Social_capital+Tuition:Foreign_born+Seg_racial:Income+Population:Local_gov_spending+Commute:HS_dropout+Graduation:Teenage_labor+Middle_class:Graduation+Gini_99:Married+Longitude:Latitude+Migration_in:Social_capital+Seg_income:Gini_99+Gini:HS_dropout+Graduation:Manufacturing+Tuition:Social_capital+Seg_poverty:Colleges+School_spending:Manufacturing+Seg_poverty:Longitude+Income:Tuition+School_spending:Graduation+School_spending:Test_scores+Local_tax_rate:Single_mothers+Seg_affluence:EITC+Gini_99:Test_scores+Gini:Progressivity+Social_capital:Longitude+Commute:Student_teacher_ratio+Progressivity:Divorced+Colleges:Chinese_imports+Seg_income:Local_gov_spending+Income:Manufacturing+Seg_affluence:Teenage_labor+HS_dropout:Foreign_born+Religious:Latitude+Divorced:Latitude+Single_mothers:Married+Seg_poverty:Commute+School_spending:Teenage_labor+Teenage_labor:Religious+Seg_poverty:Share01+Local_tax_rate:Tuition,data=dataset)


model7 = lm(Mobility~.+Seg_affluence:Latitude+Seg_racial:Single_mothers+Commute:Migration_out+Share01:Local_gov_spending+EITC:Social_capital+Test_scores:Latitude+Seg_affluence:Gini+Black:Seg_poverty+Student_teacher_ratio:Graduation+Local_gov_spending:Religious+Commute:Foreign_born+Labor_force_participation:Married+Progressivity:Social_capital+Progressivity:Migration_out+Seg_income:Latitude+Single_mothers:Latitude+Seg_poverty:Local_gov_spending+Seg_affluence:HS_dropout+Religious:Single_mothers+Seg_income:School_spending+Labor_force_participation:Migration_out+Black:Seg_affluence+Gini_99:Colleges+Population:Religious+Graduation:Latitude+HS_dropout:Migration_in+Graduation:Foreign_born+Local_tax_rate:Labor_force_participation+EITC:Married+Income:Middle_class+Student_teacher_ratio:Divorced+Black:Colleges+Foreign_born:Divorced+HS_dropout:Violent_crime+Seg_racial:Test_scores+Social_capital:Single_mothers+Local_gov_spending:Test_scores+Violent_crime:Latitude+Population:Migration_out+Black:Seg_income+Commute:Labor_force_participation+Seg_racial:School_spending+Gini_99:Social_capital+Local_tax_rate:Progressivity+Commute:Religious+Income:Migration_in+School_spending:Colleges+Tuition:Longitude+Student_teacher_ratio:Manufacturing+Local_tax_rate:Social_capital+Population:Middle_class+Student_teacher_ratio:Chinese_imports+Labor_force_participation:Migration_in+Gini:Religious+Seg_racial:Married+Share01:Local_tax_rate+Population:Graduation+Commute:Divorced+Seg_racial:Migration_out+Seg_poverty:Income+Population:Black+Middle_class:Divorced+Population:Divorced+Seg_poverty:Manufacturing+Seg_poverty:Teenage_labor+Tuition:Single_mothers+Seg_affluence:Share01+Migration_in:Latitude+Local_gov_spending:Single_mothers+HS_dropout:Manufacturing+Share01:Migration_in+Middle_class:Religious+Share01:Tuition+Share01:Test_scores+Seg_poverty:Chinese_imports+Local_gov_spending:EITC+Seg_income:Income+Population:Married+Divorced:Married+School_spending:Migration_out+Black:Married+Population:Violent_crime+Tuition:Married+Migration_in:Religious+Share01:School_spending+Student_teacher_ratio:Foreign_born+HS_dropout:Migration_out+Tuition:Graduation+Income:Single_mothers+Colleges:Teenage_labor+Seg_poverty:Divorced+Graduation:Migration_out+Seg_affluence:Local_gov_spending+Income:Colleges+Gini_99:Local_tax_rate+Commute:Colleges+Income:HS_dropout+Middle_class:Foreign_born+Income:Graduation+Seg_poverty:Married,data=dataset)


model8 = lm(Mobility~.+Student_teacher_ratio:Labor_force_participation+Share01:Single_mothers+School_spending:Migration_in+Seg_affluence:Religious+Seg_poverty:School_spending+Religious:Longitude+Graduation:Married+Population:Commute+Seg_income:Divorced+Gini:Graduation+Black:Chinese_imports+Local_tax_rate:Foreign_born+Seg_racial:Religious+Student_teacher_ratio:Social_capital+Population:Teenage_labor+Commute:School_spending+Gini:EITC+Local_tax_rate:Manufacturing+HS_dropout:Single_mothers+Local_tax_rate:Chinese_imports+Religious:Divorced+Migration_out:Married+Middle_class:Labor_force_participation+Progressivity:Latitude+Labor_force_participation:Religious+Gini_99:Latitude+Seg_racial:Commute+Population:Seg_racial+Progressivity:School_spending+Local_tax_rate:Local_gov_spending+Commute:Teenage_labor+Income:Divorced+Seg_income:Single_mothers+Gini:Violent_crime+Test_scores:Teenage_labor+Seg_racial:Colleges+Local_tax_rate:EITC+Local_tax_rate:Test_scores+Income:Progressivity+Income:Gini_99+Population:Seg_income+Local_tax_rate:Latitude+Share01:Chinese_imports+Seg_income:Religious+Test_scores:Foreign_born+Income:Student_teacher_ratio+Teenage_labor:Latitude+Gini:Colleges+Commute:Test_scores+Middle_class:Test_scores+Seg_income:Test_scores+Single_mothers:Longitude+Seg_income:Seg_affluence+Seg_affluence:Divorced+Seg_racial:Longitude+Commute:Gini_99+Seg_affluence:Middle_class+Seg_poverty:Local_tax_rate+Tuition:Labor_force_participation+Colleges:Violent_crime+Share01:Graduation+Local_gov_spending:Progressivity+Commute:Share01+Tuition:Migration_in+Manufacturing:Married+Gini_99:Chinese_imports+Gini_99:HS_dropout+Gini_99:Longitude+Population:Gini_99+Graduation:Religious+Income:Social_capital+Labor_force_participation:Foreign_born+School_spending:Student_teacher_ratio+Gini:Tuition+Black:Migration_out+HS_dropout:Longitude+Graduation:Divorced+Progressivity:Migration_in+Progressivity:HS_dropout+Gini_99:Divorced+Seg_racial:Gini_99+EITC:Single_mothers+Seg_income:Longitude+Population:Progressivity+Population:Gini+Gini_99:Teenage_labor+EITC:Migration_out+Black:Religious+Middle_class:Local_gov_spending+Share01:Progressivity+Seg_poverty:Progressivity+Local_tax_rate:Student_teacher_ratio+Progressivity:Manufacturing+Seg_affluence:Labor_force_participation+Middle_class:Longitude+Black:Divorced+School_spending:Single_mothers+Manufacturing:Teenage_labor+HS_dropout:Labor_force_participation+Commute:Middle_class+Gini:Teenage_labor,data=dataset)


model9 = lm(Mobility~.+Religious:Married+School_spending:Social_capital+Progressivity:EITC+Share01:Divorced+Gini_99:Manufacturing+Seg_income:Tuition+Commute:EITC+Seg_income:Chinese_imports+Married:Longitude+Middle_class:Married+Seg_affluence:Violent_crime+Divorced:Longitude+Seg_poverty:Gini+EITC:Teenage_labor+Manufacturing:Foreign_born+HS_dropout:Chinese_imports+Test_scores:Tuition+Test_scores:Migration_out+Seg_racial:Seg_poverty+EITC:Migration_in+Black:School_spending+Black:Progressivity+Colleges:Married+Gini:Single_mothers+Teenage_labor:Single_mothers+Black:Graduation+Student_teacher_ratio:Colleges+Test_scores:Colleges+Gini_99:Graduation+Seg_racial:Divorced+Population:HS_dropout+Chinese_imports:Migration_in+Seg_affluence:School_spending+Tuition:Chinese_imports+Seg_racial:Social_capital+Manufacturing:Migration_in+Chinese_imports:Teenage_labor+Colleges:Longitude+Labor_force_participation:Teenage_labor+Population:Seg_poverty+Migration_out:Divorced+Manufacturing:Migration_out+HS_dropout:Teenage_labor+Seg_affluence:Gini_99+Progressivity:Colleges+Foreign_born:Single_mothers+Middle_class:EITC+Foreign_born:Social_capital+Population:Colleges+Test_scores:Divorced+Student_teacher_ratio:Migration_in+Share01:Student_teacher_ratio+Local_gov_spending:Chinese_imports+Violent_crime:Longitude+Seg_income:Local_tax_rate+Colleges:Labor_force_participation+Foreign_born:Violent_crime+Teenage_labor:Married+Commute:Gini+School_spending:Labor_force_participation+Commute:Longitude+Seg_income:Married+Black:Latitude+Seg_income:Colleges+Population:Student_teacher_ratio+Manufacturing:Religious+Progressivity:Test_scores+Black:Income+Teenage_labor:Migration_in+Chinese_imports:Married+Colleges:Tuition+Chinese_imports:Violent_crime+Black:Foreign_born+Graduation:Single_mothers+Seg_poverty:Test_scores+Seg_racial:Seg_income+Share01:HS_dropout+Seg_racial:Violent_crime+Colleges:Religious+Tuition:Manufacturing+Teenage_labor:Longitude+Seg_income:Graduation+Labor_force_participation:Latitude+Black:Local_gov_spending+Teenage_labor:Divorced+Population:Share01+Gini:School_spending+Middle_class:Latitude+Commute:Latitude+Student_teacher_ratio:Married+Black:Seg_racial+Migration_in:Longitude+EITC:Longitude+Local_gov_spending:Foreign_born+Share01:Longitude+Share01:Migration_out+Student_teacher_ratio:Tuition+Seg_poverty:Tuition+Violent_crime:Divorced+Income:Longitude+Black:Local_tax_rate,data=dataset)


model10 = lm(Mobility~.+Test_scores:Longitude+Gini:Latitude+EITC:Latitude+Gini:Manufacturing+Student_teacher_ratio:Latitude+EITC:Tuition+Population:Chinese_imports+Gini_99:EITC+Migration_in:Foreign_born+Foreign_born:Religious+Chinese_imports:Social_capital+Married:Latitude+Gini:Local_tax_rate+Local_gov_spending:Teenage_labor+Gini_99:School_spending+Gini_99:Migration_out+Manufacturing:Longitude+Social_capital:Divorced+Middle_class:Progressivity+Progressivity:Teenage_labor+Test_scores:Labor_force_participation+Local_gov_spending:Migration_out+Seg_income:Student_teacher_ratio+Seg_racial:Local_tax_rate+Seg_income:Foreign_born+Income:Migration_out+Income:Violent_crime+Population:School_spending+Seg_racial:Middle_class+Religious:Violent_crime+Black:Violent_crime+Income:Teenage_labor+Graduation:Migration_in+Seg_income:Manufacturing+Share01:Middle_class+Share01:Manufacturing+Teenage_labor:Social_capital+Seg_poverty:Seg_affluence+Local_tax_rate:Migration_in+Share01:Gini_99+Seg_income:Migration_in+Population:Migration_in+Migration_out:Single_mothers+Commute:Violent_crime+Social_capital:Latitude+HS_dropout:Social_capital+Local_gov_spending:Social_capital+Share01:Colleges+Income:Local_tax_rate+EITC:Labor_force_participation+Local_tax_rate:Married+Test_scores:HS_dropout+Teenage_labor:Foreign_born+Progressivity:Foreign_born+Gini_99:Middle_class+Progressivity:Graduation+Tuition:Teenage_labor+Chinese_imports:Single_mothers+Student_teacher_ratio:Teenage_labor+Chinese_imports:Migration_out+Single_mothers:Divorced+Labor_force_participation:Manufacturing+Black:Longitude+Middle_class:Violent_crime+Tuition:Religious+Middle_class:Social_capital+Graduation:Violent_crime+Seg_racial:Teenage_labor+Middle_class:Single_mothers+Seg_income:EITC+Black:Manufacturing+Seg_poverty:Latitude+HS_dropout:Married+Commute:Married+Commute:Local_gov_spending+Local_gov_spending:Latitude+Seg_affluence:Longitude+Local_gov_spending:Labor_force_participation+Seg_affluence:Student_teacher_ratio+HS_dropout:Religious+Seg_poverty:Social_capital+EITC:Divorced+Middle_class:Local_tax_rate+Chinese_imports:Latitude+Gini_99:Religious+School_spending:HS_dropout+Middle_class:Migration_in+Share01:Married+Seg_income:Teenage_labor+Black:HS_dropout+Gini:Social_capital+Seg_affluence:Colleges+Commute:Social_capital+Population:EITC+Middle_class:Colleges+Seg_income:Social_capital+EITC:Colleges+Local_tax_rate:School_spending+Income:School_spending+Middle_class:Migration_out+Seg_income:Seg_poverty,data=dataset)




# Only significant ones

model11 = lm(Mobility~.+Middle_class:Teenage_labor+Gini_99:Violent_crime+Black:Test_scores+Seg_affluence:Progressivity+Seg_affluence:Migration_in+Labor_force_participation:Longitude+Seg_poverty:EITC+Student_teacher_ratio:Longitude+School_spending:Religious+Local_tax_rate:Teenage_labor+Seg_poverty:Middle_class+Colleges:Latitude+Tuition:Migration_out+Gini:Divorced+Seg_poverty:HS_dropout+Local_tax_rate:Colleges+Seg_affluence:Local_tax_rate+Seg_racial:HS_dropout+Gini:Local_gov_spending+Progressivity:Religious+Middle_class:HS_dropout+Tuition:Latitude+Commute:Progressivity+Seg_racial:Progressivity+Student_teacher_ratio:Religious+Middle_class:Manufacturing+HS_dropout:Divorced+Test_scores:Religious+Seg_income:Middle_class+Migration_in:Migration_out+Foreign_born:Latitude+Migration_in:Social_capital+School_spending:Test_scores+Gini_99:Test_scores+Commute:Student_teacher_ratio+Commute:Migration_out+Test_scores:Latitude+Commute:Foreign_born+Gini_99:Colleges+Commute:Labor_force_participation+Local_tax_rate:Progressivity+Commute:Religious+Local_tax_rate:Social_capital+Seg_racial:Married+Share01:Test_scores+Income:Single_mothers+Income:Colleges+Middle_class:Foreign_born+Seg_poverty:Married+HS_dropout:Single_mothers+Religious:Divorced+Gini_99:Latitude+Progressivity:School_spending+Seg_racial:Colleges+Commute:Test_scores+Gini_99:HS_dropout+Income:Social_capital+Gini_99:Divorced+Population:Progressivity+Seg_poverty:Progressivity+Black:Divorced+Commute:Middle_class+Gini_99:Manufacturing+HS_dropout:Chinese_imports+Labor_force_participation:Teenage_labor+Progressivity:Colleges+Foreign_born:Social_capital+Seg_racial:Seg_income+Share01:HS_dropout+Middle_class:Latitude+Test_scores:Longitude+Gini:Latitude+Seg_income:Manufacturing+Test_scores:HS_dropout+Progressivity:Graduation+Middle_class:Violent_crime+Middle_class:Single_mothers+HS_dropout:Religious+Black:HS_dropout+Gini:Social_capital+Population:EITC+EITC:Colleges,data=dataset)

# See if we are not estimating too many parameters
nrow(dataset)/3 >length(coef(model11))

# Lets tag this model
secondModel = model11
```

3. Final take at improvement

```{r,warning=FALSE,message=FALSE} 
mod_both_aic = step(model3,model11,direction = "both",trace=0)
                     
# Tag this too.
thirdModel = mod_both_aic

```

4. Transformations?


```{r,warning=FALSE,message=FALSE}
par(mfrow=c(1,3))
boxcox(firstModel, plotit = TRUE)
boxcox(secondModel, plotit = TRUE)
boxcox(thirdModel, plotit = TRUE)

# Log Transforms
firstLog = lm(log(Mobility)~Seg_income+Seg_poverty+Commute+Income+Gini+Share01+Middle_class+Progressivity+EITC+School_spending+HS_dropout+Colleges+Labor_force_participation+Manufacturing+Social_capital+Single_mothers+Longitude+Latitude+I(Seg_racial^2)+I(Commute^2)+I(Gini_99^2)+I(Middle_class^2)+I(School_spending^2)+I(HS_dropout^2)+I(Social_capital^2)+I(Religious^2)+I(Longitude^2)+I(Latitude^2),data=dataset)
secondLog = lm(log(Mobility)~.+Middle_class:Teenage_labor+Gini_99:Violent_crime+Black:Test_scores+Seg_affluence:Progressivity+Seg_affluence:Migration_in+Labor_force_participation:Longitude+Seg_poverty:EITC+Student_teacher_ratio:Longitude+School_spending:Religious+Local_tax_rate:Teenage_labor+Seg_poverty:Middle_class+Colleges:Latitude+Tuition:Migration_out+Gini:Divorced+Seg_poverty:HS_dropout+Local_tax_rate:Colleges+Seg_affluence:Local_tax_rate+Seg_racial:HS_dropout+Gini:Local_gov_spending+Progressivity:Religious+Middle_class:HS_dropout+Tuition:Latitude+Commute:Progressivity+Seg_racial:Progressivity+Student_teacher_ratio:Religious+Middle_class:Manufacturing+HS_dropout:Divorced+Test_scores:Religious+Seg_income:Middle_class+Migration_in:Migration_out+Foreign_born:Latitude+Migration_in:Social_capital+School_spending:Test_scores+Gini_99:Test_scores+Commute:Student_teacher_ratio+Commute:Migration_out+Test_scores:Latitude+Commute:Foreign_born+Gini_99:Colleges+Commute:Labor_force_participation+Local_tax_rate:Progressivity+Commute:Religious+Local_tax_rate:Social_capital+Seg_racial:Married+Share01:Test_scores+Income:Single_mothers+Income:Colleges+Middle_class:Foreign_born+Seg_poverty:Married+HS_dropout:Single_mothers+Religious:Divorced+Gini_99:Latitude+Progressivity:School_spending+Seg_racial:Colleges+Commute:Test_scores+Gini_99:HS_dropout+Income:Social_capital+Gini_99:Divorced+Population:Progressivity+Seg_poverty:Progressivity+Black:Divorced+Commute:Middle_class+Gini_99:Manufacturing+HS_dropout:Chinese_imports+Labor_force_participation:Teenage_labor+Progressivity:Colleges+Foreign_born:Social_capital+Seg_racial:Seg_income+Share01:HS_dropout+Middle_class:Latitude+Test_scores:Longitude+Gini:Latitude+Seg_income:Manufacturing+Test_scores:HS_dropout+Progressivity:Graduation+Middle_class:Violent_crime+Middle_class:Single_mothers+HS_dropout:Religious+Black:HS_dropout+Gini:Social_capital+Population:EITC+EITC:Colleges,data=dataset)
thirdLog = lm(log(Mobility)~Seg_income+Seg_poverty+Commute+Income+Gini+Share01+Middle_class+Progressivity+EITC+School_spending+HS_dropout+Colleges+Labor_force_participation+Manufacturing+Social_capital+Single_mothers+Longitude+Latitude+I(Seg_racial^2)+
+I(Commute^2)+I(Gini_99^2)+I(Middle_class^2)+I(School_spending^2)+I(HS_dropout^2)+I(Social_capital^2)+I(Religious^2)+I(Longitude^2)+I(Latitude^2),data=dataset)


```

### Model Diagnostics

We have Six models to dignose: `firstModel`,`secondModel` and `thirdModel` and their transformed `log` versions.

#### Model Assumptions

1. Linearity and Constant Variance

```{r,warning=FALSE,message=FALSE}
par(mfrow=c(1,3))
plot_fitted_resid(firstModel)
plot_fitted_resid(secondModel)
plot_fitted_resid(thirdModel)

par(mfrow=c(1,3))
plot_fitted_resid(firstLog)
plot_fitted_resid(secondLog)
plot_fitted_resid(thirdLog)

bptest(firstModel)
bptest(secondModel)
bptest(thirdModel)

bptest(firstLog)
bptest(secondLog)
bptest(thirdLog)
```

Based on these tests and plots, `secondLog` is the winner.

2. Normailty of errors

```{r,warning=FALSE,message=FALSE}
par(mfrow=c(3,2))
hist(resid(firstModel),xlab = "Residuals",,main = "Histogram of Residuals, firstModel",col = "darkorange",border = "dodgerblue")
plot_qq(firstModel)
hist(resid(secondModel),xlab = "Residuals",,main = "Histogram of Residuals, secondModel",col = "darkorange",border = "dodgerblue")
plot_qq(secondModel)
hist(resid(thirdModel),xlab = "Residuals",,main = "Histogram of Residuals, thirdModel",col = "darkorange",border = "dodgerblue")
plot_qq(thirdModel)

par(mfrow=c(3,2))
hist(resid(firstLog),xlab = "Residuals",,main = "Histogram of Residuals, Log irstModel",col = "darkorange",border = "dodgerblue")
plot_qq(firstLog)
hist(resid(secondLog),xlab = "Residuals",,main = "Histogram of Residuals, Log secondModel",col = "darkorange",border = "dodgerblue")
plot_qq(secondLog)
hist(resid(thirdLog),xlab = "Residuals",,main = "Histogram of Residuals, Log thirdModel",col = "darkorange",border = "dodgerblue")
plot_qq(thirdLog)

shapiro.test(resid(firstModel))
shapiro.test(resid(secondModel))
shapiro.test(resid(thirdModel))

shapiro.test(resid(firstLog))
shapiro.test(resid(secondLog))
shapiro.test(resid(thirdLog))
```
`secondLog` model is winner based on plots and tests.

#### Unsual Observations

```{r,warning=FALSE,message=FALSE}
# Leverage
length(hatvalues(firstModel)[hatvalues(firstModel) > 2 * mean(hatvalues(firstModel))])
length(hatvalues(secondModel)[hatvalues(secondModel) > 2 * mean(hatvalues(secondModel))])
length(hatvalues(thirdModel)[hatvalues(thirdModel) > 2 * mean(hatvalues(thirdModel))])

length(hatvalues(firstLog)[hatvalues(firstLog) > 2 * mean(hatvalues(firstLog))])
length(hatvalues(secondLog)[hatvalues(secondLog) > 2 * mean(hatvalues(secondLog))])
length(hatvalues(thirdLog)[hatvalues(thirdLog) > 2 * mean(hatvalues(thirdLog))])

# Outliers
length(rstandard(firstModel)[abs(rstandard(firstModel)) > 2])
length(rstandard(secondModel)[abs(rstandard(secondModel)) > 2])
length(rstandard(thirdModel)[abs(rstandard(thirdModel)) > 2])


length(rstandard(firstLog)[abs(rstandard(firstLog)) > 2])
length(rstandard(secondLog)[abs(rstandard(secondLog)) > 2])
length(rstandard(thirdLog)[abs(rstandard(thirdLog)) > 2])


# Influential
length(cooks.distance(firstModel)[cooks.distance(firstModel) > 4 / length(cooks.distance(firstModel))])
length(cooks.distance(secondModel)[cooks.distance(secondModel) > 4 / length(cooks.distance(secondModel))])
length(cooks.distance(thirdModel)[cooks.distance(thirdModel) > 4 / length(cooks.distance(thirdModel))])

length(cooks.distance(firstLog)[cooks.distance(firstLog) > 4 / length(cooks.distance(firstLog))])
length(cooks.distance(secondLog)[cooks.distance(secondLog) > 4 / length(cooks.distance(secondLog))])
length(cooks.distance(thirdLog)[cooks.distance(thirdLog) > 4 / length(cooks.distance(thirdLog))])


```
Prsence of Outliers, Influential points and Leveraging points is expected in real life datasets. 

#### Evaluations
```{r,warning=FALSE,message=FALSE}
summary(firstModel)$adj.r.squared 
summary(secondModel)$adj.r.squared 
summary(thirdModel)$adj.r.squared 
summary(firstLog)$adj.r.squared 
summary(secondLog)$adj.r.squared 
summary(thirdLog)$adj.r.squared 


calc_loocv_rmse(firstModel)
calc_loocv_rmse(secondModel)
calc_loocv_rmse(thirdModel)
calc_loocv_rmse(firstLog)
calc_loocv_rmse(secondLog)
calc_loocv_rmse(thirdLog)

# VIF
sum(vif(firstModel)>5)/length(coef(firstModel))
sum(vif(secondModel)>5)/length(coef(secondModel))
sum(vif(thirdModel)>5)/length(coef(thirdModel))
sum(vif(firstLog)>5)/length(coef(firstLog))
sum(vif(secondLog)>5)/length(coef(secondLog))
sum(vif(thirdLog)>5)/length(coef(thirdLog))

#AIC
extractAIC(firstModel)
extractAIC(secondModel)
extractAIC(thirdModel)
extractAIC(firstLog)
extractAIC(secondLog)
extractAIC(thirdLog)

#BIC
extractAIC(firstModel,k=log(nrow(dataset)))
extractAIC(secondModel,k=log(nrow(dataset)))
extractAIC(thirdModel,k=log(nrow(dataset)))
extractAIC(firstLog,k=log(nrow(dataset)))
extractAIC(secondLog,k=log(nrow(dataset)))
extractAIC(thirdLog,k=log(nrow(dataset)))
```

Depending upon our criteria, we have different winners. For `adj.r.squared`, its `secondLog` and for `LOOCV_RMSE`, its tie between `firstModel` and `thirdModel`. And even after considering 2-Way interactions; we still have multicollinearity issues.
<br/>
#### Plots
```{r,warning=FALSE,message=FALSE}
par(mfrow = c(2, 2))
plot(firstModel,main="firstModel")
par(mfrow = c(2, 2))
plot(secondModel,main="secondModel")
par(mfrow = c(2, 2))
plot(thirdModel,main="thirdModel")

par(mfrow = c(2, 2))
plot(firstLog,main="Log firstModel")
par(mfrow = c(2, 2))
plot(secondLog,main="Log secondModel")
par(mfrow = c(2, 2))
plot(thirdLog,main="Log thirdModel")
```

## Results

We chose the `secondLog` model as our final one, even of its issues with multicollinearity, its still the best in terms of `Adjusted R2` value.

Lets see which predictors we finally have.

```{r,warning=FALSE,message=FALSE}
length(coef(secondLog))
names(coef(secondLog))
```

We find that its a highly interactive model. And looking at the significance level of each contributor we find that there's still hige scope for improvement. Lets do that one last time.

```{r,warning=FALSE,message=FALSE}
finalModel = step(secondLog,trace=0)
```

And see how it performs.

```{r,warning=FALSE,message=FALSE}
par(mfrow = c(2, 2))
plot(finalModel,main="Final Model")

# Evaluation
summary(finalModel)$adj.r.squared 
summary(finalModel)$adj.r.squared > summary(secondLog)$adj.r.squared 

calc_loocv_rmse(finalModel)
calc_loocv_rmse(finalModel)<calc_loocv_rmse(secondLog)

# VIF
sum(vif(finalModel)>5)/length(coef(finalModel))
sum(vif(finalModel)>5) < sum(vif(secondLog)>5)

#AIC
extractAIC(finalModel)
extractAIC(secondLog)


#BIC
extractAIC(finalModel,k=log(nrow(dataset)))
extractAIC(secondLog,k=log(nrow(dataset)))
```

We found a better performing model :)

## Discussion

With the `finalModel` we have some observations.

*`Commute` which we considered to be the most important predictor of economic mobility, is not significant considering the interactions.*

```{r,warning=FALSE,message=FALSE}
coef(summary(finalModel))['Commute',]
```

*The significant predictors.*

```{r}
sum(summary(finalModel)$coefficients[ ,4] < 0.05)/length(coef(finalModel))
summary(finalModel)$coefficients[summary(finalModel)$coefficients[ ,4] < 0.05,]
```

*Proportion of variation in `Mobility` explained by chosen predictors.*
```{r,warning=FALSE,message=FALSE}
summary(finalModel)$r.squared 
```
*Really a better model than `secondLog`*
```{r,warning=FALSE,message=FALSE}
analysis = anova(finalModel,secondLog,test="F")
analysis$`Pr(>F)`[2]
```

YES!! We fail to reject the NULL hypothesis that the smaller model is as good as the bigger one.

## Appendix

**Model Summaries**
<br/>
*secondLog*

```{r,warning=FALSE,message=FALSE}
summary(secondLog)
```

*finalModel*

```{r,warning=FALSE,message=FALSE}
summary(finalModel)
```

**A model with `States`**


```{r,warning=FALSE,message=FALSE}

  # Dropping unique IDs and Names; and also States (too many levels.)
  
  drops <- c("Name", "ID")
  dataset = mobilityData[ , !(names(mobilityData) %in% drops)]
  
  # Lets see interactions in a tree model
  form <- as.formula(Mobility ~ .)
  model <- rpart(form,data=dataset)
  prp(model)
  
  # Correlations
  data = na.omit(mobility)
  #round(cor(data[sapply(data,is.numeric)], use="pairwise.complete.obs"),2)
  corrplot(cor(data[sapply(data,is.numeric)]),method ="ellipse",
         title =" Correlation Matrix Graph",tl.cex = .5,tl.pos ="lt",tl.col ="dodgerblue" )
  
  
  # State model
   modelState = lm(Mobility~Population+Black+Urban+State+Seg_racial+Seg_income+Seg_poverty+Seg_affluence+Commute+Income+Gini+Share01+Gini_99+Middle_class+Local_tax_rate+Local_gov_spending+Progressivity+EITC+School_spending+Student_teacher_ratio+Test_scores+HS_dropout+Colleges+Tuition+Graduation+Labor_force_participation+Manufacturing+Chinese_imports+Teenage_labor+Migration_in+Migration_out+Foreign_born+Social_capital+Religious+Violent_crime+Single_mothers+Divorced+Married+Longitude+Latitude+I(Population^2)+I(Black^2)+I(Seg_racial^2)+I(Seg_income^2)+I(Seg_poverty^2)+I(Seg_affluence^2)+I(Commute^2)+I(Income^2)+I(Gini^2)+I(Share01^2)+I(Gini_99^2)+I(Middle_class^2)+I(Local_tax_rate^2)+I(Local_gov_spending^2)+I(Progressivity^2)+I(EITC^2)+I(School_spending^2)+I(Student_teacher_ratio^2)+I(Test_scores^2)+I(HS_dropout^2)+I(Colleges^2)+I(Tuition^2)+I(Graduation^2)+I(Labor_force_participation^2)+I(Manufacturing^2)+I(Chinese_imports^2)+I(Teenage_labor^2)+I(Migration_in^2)+I(Migration_out^2)+I(Foreign_born^2)+I(Social_capital^2)+I(Religious^2)+I(Violent_crime^2)+I(Single_mothers^2)+I(Divorced^2)+I(Married^2)+I(Longitude^2)+I(Latitude^2),data=dataset)
   
# Significant ones
 summary(modelState)$coefficients[summary(modelState)$coefficients[ ,4] < 0.05,]
  
```  
Only Two states are significant; thus we were okay dropping them in the beginning.

