

###Introduction###<br>

An older female applicant for a job reckons her failure to be hired as a teacher is due to her age. So she accuses the school of illegally favoring younger (i.e. under 40 years old ) applicants. The school, on the other side, contradicts by saying that the differences of admission among agegroups can be explained by other characteristics of applicants. A dataset collected from school files is provided. The dataset presents 631 applicants' infomartion of 15 variables: 1.) interviewed: yes/no vairable of whether the applicant is interviewed; 2.) hired: yes/no vairable of whether the applicant is hired; 
3.) appdate: application date; 4.) age: age of the applicant; 5.) sex: female/male; 6.) residence:taking values in 1/2/3/4, where 1 = "lives in the same town", 2 = "lives in the state but not in this town", 3 = "lives out of state", and 4 = "lives out of country"; 7.) GPA.u: undergraduate grade point averages; 8.) GPA.g: graduate grade point averages; 9.) MA: taking values in yes/no/pending, indicating whether or not the applicant has a relevant (pending) master's degree; 10.) subsititute: yes/no variable indicating whether the applicant has prior experience as a substitute teacher; 11.) teaching: yes/no variable indicating whether the applicant has prior experience as a full-time teacher; 12.) experience: the time span of relevant experience; 13.) workkids: yes/no variable indicating whether the applicant has experience working with kids in the past; 14.) volunter: yes/no variable indicating whether the applicant has experience as volunteers in the past; 15.) agegroup: older/ younger corresponding to age, if age is 40 years or older, count it as older agegroup and younger otherwise.

###Data Processing###<br>

Several steps are taken to ensure that the data containsonly representations of "typical" applicants in order to rule out other possibilities that could influence the job admission. First, there are 65 out of 631 applicants with missing values in the dataset, as shown in the plot, mostly lacking agegroup and age information, so these applicants are removed. Also, 17 entries of "N/A" or "" (blank) within variables of teaching, MA, experience,and subsititute are removed. However, there are 351(55.63%) "N/A" entries in GPA.u and GPA.g, which cannot be removed directly and then ignored. So we select those applicants with available GPA.u and GPA.g as the training dataset to see if the two variables have significant influence. Next, there are confusing entries of experience like 0.11 and 3 months(instead of years) that are higly likely due to data entry errors, since this is only a small fraction(~2.22%), just remove them. Finally, formatting problem is manipulated by deleting unnecessary whitespaces and asterisks. 
```r
x <- read.csv("TeacherHires.csv", as.is=TRUE)
library(Amelia)
missmap(x, main = "Missing values vs. observed data")
```

###Methods and Exploration###<br>

In order to investigate the influence of age on recruitment results, several approaches are considered. To begin with, the simple  chi-square test is used to test the association between the row and column variables in a two-way table. 
```r
x <- na.omit(x)[,-c(1,3)]
x$MA <- gsub("^\\s+|\\s+$", "", x$MA)# Trim whitespace from both start and end 
x$hired <- gsub("^\\s+|\\s+$|[*]", "", x$hired)
x$substitute <- gsub("^\\s+|\\s+$", "", x$substitute)
x$teaching <- gsub("^\\s+|\\s+$", "", x$teaching)
x$workkids <- gsub("^\\s+|\\s+$", "", x$workkids)
x$GPA.g <- gsub("^\\s+|\\s+$", "", x$GPA.g)
# chi-square
tab <- table(x$hired, x$agegroup);tab
chisq.test(tab)
```
The p-value 0.02579 indicates that some association between recruitment results and agegroups is present, though it is not that obvious from the distribution of age for hired and non-hired applicants.

```{r, echo=FALSE, fig.height=13, fig.width=24, dpi=200, message=FALSE, warning=FALSE}
par(mfrow=c(1,2))
par(cex=2.5)
library(MASS)
truehist(x$age[x$hired=="yes"],xlim = range(x$age))
title("Distribution of age (hired applicants) ")
truehist(x$age[x$hired=="no"],xlim = range(x$age))
title("Distribution of age (non-hired applicants) ")
par(mfrow=c(1,1))
```
Further research is whether agegroup affects the hired results when all the variables are taken into consideration. Application dates and whether interviewed or not are not characteristics of an applicant in nature, so they are taken out.

Logistic regression allows us to fit a categorical variable by a given set of predictors x, which can be continuous, categorical or a mix of both.


```r
ind <- which(x$substitute == ""|x$teaching == " " )
x <- x[-ind, ]
ind <- which(x$teaching == "N/A"| x$MA == "N/A" | x$experience == "N/A")
x <- x[-ind, ]
# We cannot make sure if there is data entry error for experience like 0.11 and 3 months, since this is only a small fraction, we just delete it.
ind <- grep("\\D", x$experience)
x <- x[-ind, ]
x[,c(1,8,9,11,12)] <- ifelse(x[,c(1,8,9,11,12)] =="yes", 1, 0)
x$MA <- as.numeric(as.factor(x$MA))
x$sex <- ifelse(x$sex == "Male", 1, 0)
x$agegroup <- ifelse(x$agegroup == "younger", 1, 0)
ind <- which(x$GPA.u == "N/A"| x$GPA.g == "N/A" |x$GPA.g == ""  )
# train data first with available GPA.u and GPA.g
train <- data.frame(apply(as.matrix(x[-ind, ]), 2, as.numeric))
model00 <- glm( hired ~ ., data = train, 
             family = "binomial")
summary(model00)
```
Since only residence is statistically significant from this model, and by common sense, it should be highly influential. So filtering by residence first is sensible and discard the only entry for residence = 4 .Moreover, GPA.u and GPA.g are not statistically significant, so delete the two variables (together with their missing data) and then create three models in terms of residence with randomly sampled 500 applicants as training set, while the rest as testing set.

###Results###<br>
```r
set.seed(0)
ind <- sample(1:dim(x)[1], 500, replace = FALSE)
train <- data.frame(apply(as.matrix(x[ind, -c(5,6)]), 2, as.numeric))
resname<-names(table(train$residence))[1:3]# delete 4 for only one observation
model <-list()
for (i in 1:(length(resname))){
  sameres<- which(train$residence == resname[i])
  model[[i]]<- glm( hired ~ age + sex + MA + substitute +teaching +                       experience + workkids + volunteer+ agegroup,                        data= train[sameres,], family = "binomial")
}

```

By classifying models according to  different residence types, three models are created:

___Model 1:___ `residence`  = 1, i.e. "lives in the same town"
```{r, echo=FALSE, message=FALSE, warning=FALSE}
summary(model[[1]])

```
Both variable 'age' and 'agegroup' are not statistically significant, nor are other predictors, so there could be other factors affecting the hiring decision that cannot be figured out in this dataset.

___Model 2:___  `residence`  = 2, i.e. "lives in the state but not in this town"
```{r, echo=FALSE, message=FALSE, warning=FALSE}
summary(model[[2]])
```
Under this scenario, agegroup has p-value of 0.096, suggesting a strong association of agegroup of the applicant with the probability of being hired. The positive coefficient for agegroup suggests that all other variables being equal, a younger applicant is more likely to be hired. 

___Model 3:___  `residence`  = 3, i.e. "lives out of state"
```r
summary(model[[3]])

```
In this situation, all predictors fail. Take a closer look at the recruiting result for applicants living out of the state, all of them are not hired. When the dependent variable has only one value, this is definitely not a good model and no clues can be detected for relation between age and hiring.

Finally, conduct a further assessment of `Model 2`  to see how it is doing when predicting hired on testing data:
```r
test <- data.frame(apply(as.matrix(x[-ind, -c(5,6)]), 2, as.numeric))
fitted.results <- predict(model[[2]], newdata=test, type='response')
fitted.results <- ifelse(fitted.results > 0.5,1,0)
misClasificError <- mean(fitted.results != test$hired)
print(paste('Accuracy',1-misClasificError))
```
The 0.886 accuracy on the test set is quite a good result, so in this sense  `Model 2` can be trusted.

As discussed above, for applicants living in the same town and living out of state, no significant bias toward older applicants are found. For applicants living in the state but not in this town, strong favor towards younger group exists.




