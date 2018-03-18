
x <- read.csv("TeacherHires.csv", as.is=TRUE)
library(Amelia)
missmap(x, main = "Missing values vs. observed data")



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



par(mfrow=c(1,2))
par(cex=2.5)
library(MASS)
truehist(x$age[x$hired=="yes"],xlim = range(x$age))
title("Distribution of age (hired applicants) ")
truehist(x$age[x$hired=="no"],xlim = range(x$age))
title("Distribution of age (non-hired applicants) ")
par(mfrow=c(1,1))




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

Since only residence is statistically significant from this model, and by common sense, it should be highly influential. So filtering by residence first is sensible and discard the only entry for residence = 4 .Moreover, GPA.u and GPA.g are not statistically significant, so delete the two variables (together with their missing data) and then create three models in terms of residence with randomly sampled 500 applicants as training set, while the rest as testing set.

###Results###<br>
set.seed(0)
ind <- sample(1:dim(x)[1], 500, replace = FALSE)
train <- data.frame(apply(as.matrix(x[ind, -c(5,6)]), 2, as.numeric))
resname<-names(table(train$residence))[1:3]# delete 4 for only one observation
model <-list()
for (i in 1:(length(resname))){
  sameres<- which(train$residence == resname[i])
  model[[i]]<- glm( hired ~ age + sex + MA + substitute +teaching +                       experience + workkids + volunteer+ agegroup,                        data= train[sameres,], family = "binomial")
}



#By classifying models according to  different residence types, three models are created:

#___Model 1:___ `residence`  = 1, i.e. "lives in the same town"
summary(model[[1]])


#Both variable 'age' and 'agegroup' are not statistically significant, nor are other predictors, so there could be other #factors affecting the hiring decision that cannot be figured out in this dataset.

#___Model 2:___  `residence`  = 2, i.e. "lives in the state but not in this town"
summary(model[[2]])

#Under this scenario, agegroup has p-value of 0.096, suggesting a strong association of agegroup of the applicant with the #probability of being hired. The positive coefficient for agegroup suggests that all other variables being equal, a younger #applicant is more likely to be hired. 

#___Model 3:___  `residence`  = 3, i.e. "lives out of state"

summary(model[[3]])


#In this situation, all predictors fail. Take a closer look at the recruiting result for applicants living out of the state, 
#all of them are not hired. When the dependent variable has only one value, this is definitely not a good model and no clues #can be detected for relation between age and hiring.

#Finally, conduct a further assessment of `Model 2`  to see how it is doing when predicting hired on testing data:

test <- data.frame(apply(as.matrix(x[-ind, -c(5,6)]), 2, as.numeric))
fitted.results <- predict(model[[2]], newdata=test, type='response')
fitted.results <- ifelse(fitted.results > 0.5,1,0)
misClasificError <- mean(fitted.results != test$hired)
print(paste('Accuracy',1-misClasificError))

#The 0.886 accuracy on the test set is quite a good result, so in this sense  `Model 2` can be trusted.



