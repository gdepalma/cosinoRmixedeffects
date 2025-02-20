# cosinoRmixedeffects
The *cosinoRmixedeffects* package can be implemented for the estimation and prediction of the a mixed-effects cosinor model for longitudinal periodic data. *cosinoRmixedeffects* package fits a mixed-effects cosinor model and utilizing the functionality of the [emmeans]((httpZs://github.com/rvlenth/emmeans)) package provides estimated marginal means (EMMs) and contrast for binary, categorical predictors and their interactions. In the example presented here, we will use a circadian rhythm data collected over several days to estimate daily changes assuming random intercept for each subject and a correlated covariance structure.  Please refer to [vignettes](https://maytesuarezfarinas.github.io/cosinoRmixedeffects/vignettes/Introduction.html) for detailed usage examples. 

#### *cosinoRmixedeffects* functionality for the Warrior Watch Study is available in the preprint:
1. [Hirten, Robert P., et al.](https://www.medrxiv.org/content/10.1101/2020.11.06.20226803v1) "Longitudinal Physiological Data from a Wearable Device Identifies SARS-CoV-2 Infection and Symptoms and Predicts COVID-19 Diagnosis." medRxiv (2020).

## Installation

```{r setup}
remotes::install_github("maytesuarezfarinas/cosinoRmixedeffects")

library(cosinoRmixedeffects)
```



```{r, include = FALSE}
knitr::opts_chunk$set(
  collapse = TRUE,
  comment = "#>",
  fig.width=8,
  fig.height=3
)
```

## About

The *cosinoRmixedeffects* package can be implemented for the estimation and prediction of the a mixed-effects cosinor model for longitudinal periodic data. *cosinoRmixedeffects* package fits a mixed-effects cosinor model and utilizing the functionality of the [emmeans]((httpZs://github.com/rvlenth/emmeans)) package provides estimated marginal means (EMMs) and contrast for binary, categorical predictors and their interactions. In the example presented here, we will use a circadian rhythm data collected over several days to estimate daily changes assuming random intercept for each subject and a correlated covariance structure.   

Cosinor models can be used to model daily circadian rhythm by estimating three components (see [Cornélissen (2014)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3991883/) for details). 

1. MESOR – a non-linear function that depends on the rhythm-adjusted mean 
2. Amplitude - half the extent of variation within a day 
3. Acrophase - the time of overall high values recurring in each day

```{r echo=FALSE, out.width="50%", fig.align='center'}
knitr::include_graphics("cosinor.png")
```
Red and green dots represent hypothetical sampling times though the day from two subjects with the same daily curve, showing that features like maximum, or range will be easily biased by the sampling time. 


Currently, it is possible to model rhythmic data in R using a *cosinor* package for only daily cross-sectional designs, limited to simple group comparisons (i.e., 2-level factors). Hypothesis testing difference between groups with more than two levels or interactions are not allowed.Therefore, we extended the COSINOR model to a mixed-effects COSINOR model, to allow for longitudinal study designs where multiple predictors can be evaluated within a single model.   

We will analyze a subset of the [Warrior Watch Study](https://www.mountsinai.org/about/covid19/warrior-watch-study) data collected by the researchers at the Icahn School of Medicine at Mount Sinai. Health care workers (n=121) were monitored by a novel smartphone App employed remotely to collect biometric data, which included heart rate variability (HRV).HRV is physiological measure of time interval between heartbeats and exhibits a 24-hour circadian pattern, where changes in this pattern can be used to identify different physiological states, including infection onset (see [Hirten.Robet.P, et al.](https://www.medrxiv.org/content/10.1101/2020.11.06.20226803v1) for details about Warrior Watch Study). 

#### *cosinoRmixedeffects* functionality for the Warrior Watch Study is available in the preprint:
1. [Hirten, Robert P., et al.](https://www.medrxiv.org/content/10.1101/2020.11.06.20226803v1) "Longitudinal Physiological Data from a Wearable Device Identifies SARS-CoV-2 Infection and Symptoms and Predicts COVID-19 Diagnosis." medRxiv (2020).



## Installation

```{r setup}
remotes::install_github("maytesuarezfarinas/cosinoRmixedeffects")
library(cosinoRmixedeffects)
```

## Usage

### Installation

```{r install the package}
remotes::install_github("maytesuarezfarinas/cosinoRmixedeffects")
library(cosinoRmixedeffects)
```

### Load the package

```{r load the package, warning=FALSE, message=FALSE}
require(emmeans)
require(lme4)
require(ggplot2)
require(reshape2)
require(dplyr)
require(limma)
```


### Load the data

The dataset db.cosinor is the data with 6 variables. 

1. participant_id is the ID of the participant
2. hrv is the heart rate variability
3. gender is the biological sex for the participants
4. T0tOT14 is COVID infection status within the first 14 days of COVID test
5. bmi_baseline_cat is the baseline BMI categories as normal weight, overweight and obese
6. Hour_of_Day is the time of the day when hrv data is measured

```{r load the data}
data(db.cosinor)
head(db.cosinor)
```

### Create cosinor parameters based on the time and cosinor period

This step calculates the rrr and sss for the cosinor model, with rrr=cos(2πt/τ),sss=sin(2πt/τ), where t is the time and τ is the period of a cycle. 
Here the time (t) is Hour of the day and the period (τ) is 24 hours.

```{r data management}
## create cosinor paramenters needed for cosinor model
db.model<-create.cosinor.param(time="Hour_of_Day", period=24, data=db.cosinor)
```


### Examine the data

```{r}
## the number of participants with COVID+ and COVID-
table(db.model[!duplicated(db.model$participant_id), c("T0toT14")])
## the number of male and female participants
table(db.model[!duplicated(db.model$participant_id), c("gender")])
## the number of normal weight, overweight and obese participants
table(db.model[!duplicated(db.model$participant_id), c("bmi_baseline_cat")])
```

Using these data, we will provide three modeling examples with *cosinoRmixedeffects*’s functionality:

1. Model 1 that fits a model for biological sex (Female vs. Male), assuming random effects for MESOR;
2. Model 2 that fits COVID-19 status (positive vs negative) and adjusts for biological sex as a covariate, assuming random effects for MESOR;
3. Model 3 that fits an interaction between biological sex and BMI, assuming random effects for MESOR, amplitude and acrophase.


### Example 1. Fit model for biological sex only, assuming random effects for MESOR

Step 1. fit mixed-effects cosinor model to estimate the effects of biological sex on HRV curves (MESOR, amplitude and acrophase) using *fit.cosinor.mixed()* function. 

The arguments "y", "random" and "data" are required, and inputs for either argument "x" or "interaction" are needed. Here, we only assumed random effects for MENSOR (random intercept) in this example.We could also assume random effects for MENSOR, amplitude and acrophase by changing the argument *random="1|participant_id"* to *random = "1+rrr+sss|participant_id"*. The default action to handle missing data is "na.action=na.omit", where it returns the object with observations removed if they contain any missing values.

```{r model 1}
## Fit Model 1
f1<-fit.cosinor.mixed(y="hrv",x="gender",random="1|participant_id", data=db.model)
```


Step 2. get the estimated mean MESOR, amplitude and acrophase by gender, and use bootstrapping to get confidence intervals for the means,with function *get.means.ci.cosinor*. 

This function uses the inverse relationship to obtain EMMs for MESOR, amplitude and acrophase.The model fit object and contrast mean formula are required for this function. The default number of simulations is 500. The more simulations we conduct, the closer distribution/probability we get to the theoretical distribution/probability.The default is to calculate 95% confidence interval, but argument *conflevel* could be specified to obtain other confidence intervals, e.g. conflevel = 0.9 for 90% confidence interval . The type of interval calculated by default is "norm", but other types are also allowed by specifying the argument *conftype*.

```{r message=FALSE}
db.means<-get.means.ci.cosinor(f1, contrast.frm="~gender",nsim=500)
db.means
## the MEAN is the EMM, with 95% confidence interval lower boundary as "2.5%" and upper boundary as "97.5%" columns. 
```


Step 3. get the estimated pairwise contrasts by gender and confidence intervals for MESOR, Amplitude and acrophase of hrv, with function *get.contrasts.ci.cosinor*. 

The model object and contrast formula are required for this function.The default is 95% confidence interval, but argument *conflevel* could be specified to obtain other confidence intervals, e.g. conflevel = 0.9 for 90% confidence interval . The type of interval calculated by default is "norm", but other types are also allowed by specifying the argument *conftype*.

```{r message=FALSE, fig.show='hide'}
db.delta<-get.contrasts.ci.cosinor(f1,contrast.frm="~gender",nsim=500)
db.delta
## boot.estimate is the estimated contrasts between groups, with its 95% confidence interval lower boundary as "2.5%" and upper boundary as "97.5%" columns. 
```


Step 4. plot mean and CI for COSINOR parameters.

```{r, message=FALSE, warning=FALSE}
## This uses the db.means object you created earlier
p<-ggplot(db.means, aes(x=gender, y=MEAN, color=gender))+
  geom_point(position = position_dodge(width=1))+
  scale_color_manual(values = c(
    Woman="forestgreen", 
    Man="orange"))+
  geom_errorbar(aes(ymin=`2.5 %`, ymax=`97.5 %`),position = position_dodge(width=1))+
  theme_bw()+
  facet_wrap(~Param, scales = "free_y") +
  guides(color=guide_legend(title="Gender"))+
  labs(x="Gender", y="Mean (CI 95%)") + 
  theme(strip.background = element_rect(color="black", fill="white", linetype=0))+
  theme(strip.text.x = element_text(size = 14))
```

Here we used the function *{stat.test.stars}* to obtain stars to indicate statistical significance levels, and this function requires db.delta and contrast formula.

```{r, message=FALSE, warning=FALSE}
require(rstatix)
stat.test<-stat.test.stars(db.delta=db.delta, contrast.frm = "~gender")
require(ggpubr)
p<-p+stat_pvalue_manual(stat.test, hide.ns = T, step.increase = 0.05, tip.length = 0.03)
p
```


Step 5. plot the COSINOR circadian curves with the function *ggplot.cosinor.lmer*.

We need the fit object, the group variable, the period of a cycle, the time variable

```{r warning=FALSE}
## ## This code creates blank axis, and relies on model fit “f1”, estimated means “db.means”, and original data “db.model” objects.
p<-ggplot.cosinor.lmer(object=f1,
                       x_str="gender",
                       period=24,
                       db.means=db.means,
                       DATA=db.model)
## This code customizes the plot
p + labs(x="Time (hours)", y="HRV SDNN, (ms)")+
  scale_color_manual(values=c(Man="orange", Woman="forestgreen"))+
  geom_hline(aes(yintercept=MESOR), linetype=2, color="black")+
  theme_bw()+
  facet_wrap(~gender, nrow = 1)+
  guides(color=guide_legend(title="Gender"))+
  geom_segment(aes(x = T_AMP, 
                   y = MESOR+0.05, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude-0.05),
               arrow = arrow(length = unit(0.15, "cm"),ends="both"), 
               size=0.5, color="gray50",linetype=1)+
  geom_segment(aes(x = 0, 
                   y =  MESOR+Amplitude+0.1, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude+0.1),
               arrow = arrow(length = unit(0.15, "cm")), size=0.5, color="gray50",linetype=1)+
  geom_line(aes(color=factor(GROUP, levels = c("Man", "Woman"))))
```



### Example 2. Fit model to estimate effect of COVID-19 status on MESOR, Amplitude and Acrophase, adjusted for gender, assuming random effects for MESOR 


Step 1. fit mixed effects cosinor model to estimate effect of COVID-19 status on MESOR, Amplitude and Acrophase, adjusted for gender. 

This is achieved by adding both gender and T0toT14 variable in the argument "x" of the function *fit.cosinor.mixed*. More covariates could be added by adding more elements into the vector of argument "x".

```{r model 2}
f2<-fit.cosinor.mixed(y="hrv",x=c("gender","T0toT14"),random="1|participant_id", data=db.model)
```


Step 2. get the estimated transformed CONSINOR mean MESOR, amplitude and acrophase by COVID-19 status, adjusted for gender, and use bootstrapping to get confidence intervals for the means

```{r, message=FALSE}
db.means<-get.means.ci.cosinor(fit=f2, contrast.frm="~T0toT14", nsim=500)
db.means
```

This is to show EMMs by groups of biological sex, adjusted for COVID-19 status, and estimates changed compared to Model 1 with no covariates adjusted.
```{r, message=FALSE}
db.means2<-get.means.ci.cosinor(fit=f2, contrast.frm="~gender", nsim=500)
db.means2
```


Step 3. get the estimated pairwise contrasts by COVID-19 status and confidence intervals for MESOR, Amplitude and acrophase  

```{r, message=FALSE, fig.show='hide'}
db.delta<-get.contrasts.ci.cosinor(fit=f2,contrast.frm="~T0toT14", nsim=500)
db.delta
```


Step 4. plot mean and CI for COSINOR parameters

```{r, warning=FALSE}
p<-ggplot(db.means, aes(x=T0toT14, y=MEAN, color=T0toT14))+
  geom_point(position = position_dodge(width=1))+
  scale_color_manual(values = c(`COVID-`="forestgreen", `COVID+`="red"))+
  geom_errorbar(aes(ymin=`2.5 %`, ymax=`97.5 %`),position = position_dodge(width=1))+
  theme_bw()+
  facet_wrap(~Param, scales = "free_y") +
  guides(color=guide_legend(title="COVID-19 status"))+
  labs(x="COVID-19 status", y="Mean (CI 95%)") + 
  theme(strip.background = element_rect(color="black", fill="white", linetype=0))+
  theme(strip.text.x = element_text(size = 14))
## The code below create the object stat.test, so you can add stars to the plot and it requires db.delta
require(rstatix)
stat.test<-stat.test.stars(db.delta=db.delta, contrast.frm = "~T0toT14")
require(ggpubr)
p<-p+stat_pvalue_manual(stat.test, hide.ns = T, step.increase = 0.05, tip.length = 0.03)
p
```

Step 5. plot the COSINOR circadian curves

```{r, warning=FALSE}
## This code creates blank axis
p<-ggplot.cosinor.lmer(object=f2,
                       x_str="T0toT14",
                       period=24,
                       db.means=db.means,
                       DATA=db.model)
## This code customizes the plot
p + labs(x="Time (hours)", y="HRV SDNN, (ms)")+
  scale_color_manual(values=c(`COVID-`="forestgreen", `COVID+`="red"))+
  geom_hline(aes(yintercept=MESOR), linetype=2, color="black")+
  theme_bw()+
  facet_wrap(~T0toT14, nrow = 1)+
  guides(color=guide_legend(title="COVID-19 status"))+
  geom_segment(aes(x = T_AMP, 
                   y = MESOR+0.05, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude-0.05),
               arrow = arrow(length = unit(0.15, "cm"),ends="both"), 
               size=0.5, color="gray50",linetype=1)+
  geom_segment(aes(x = 0, 
                   y =  MESOR+Amplitude+0.1, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude+0.1),
               arrow = arrow(length = unit(0.15, "cm")), size=0.5, color="gray50",linetype=1)+
  geom_line(aes(color=factor(GROUP, levels = c("COVID-",
                                               "COVID+"))))
```

### Example 3. Interaction model, interaction between gender and BMI category, assuming random effects for MESOR, amplitude and acrophase

Step 1. fit mixed effects cosinor model with interaction between gender and BMI category

This is achieved by adding variables of interaction terms into the argument "interaction" of the function *fit.cosinor.mixed*. The argument "x" could also be used to add other covariates in addition to interactions. 

```{r model 3}
f3<-fit.cosinor.mixed(y="hrv",
                        interaction=c("gender","bmi_baseline_cat"),
                        random="1+rrr+sss|participant_id", 
                        data=db.model)
```

Step 2. get the estimated transformed CONSINOR mean MESOR, amplitude and acrophase by BMI, after adjusting for gender, and use bootstrapping to get confidence intervals for the means

To get the estimated means across body mass index (BMI) categories within each sex, the contrast formula "~bmi_baseline_cat|gender" is written in the same syntax as *emmeans*. 

```{r, message=FALSE}
db.means<-get.means.ci.cosinor(fit=f3, contrast.frm="~bmi_baseline_cat|gender", nsim=500)
head(db.means)
```

Step 3. get the estimated pairwise contrasts and the confidence intervals for MESOR, Amp, Acro for each BMI category

```{r, message=FALSE, fig.show='hide'}
db.delta<-get.contrasts.ci.cosinor(fit=f3,contrast.frm="~bmi_baseline_cat|gender", nsim=500)
head(db.delta)
```


Step 4. plot mean and CI for COSINOR parameters

```{r, warning=FALSE, fig.height=6, fig.width=8}
db.means<-mutate(db.means, 
                 bmi_baseline_cat=strsplit2(bmi_baseline_cat_gender,",")[,1],
                 gender=strsplit2(bmi_baseline_cat_gender,",")[,2])
db.means<-mutate(db.means, bmi_baseline_cat_gender=factor(bmi_baseline_cat_gender, 
                                  levels = c("Normal,Woman", "Overweight,Woman", "Obese,Woman" ,
                                             "Normal,Man" ,  "Overweight,Man", "Obese,Man")),
                 bmi_baseline_cat=factor(bmi_baseline_cat, 
                                  levels = c("Normal", "Overweight", "Obese")))
p<-ggplot(db.means, aes(x=bmi_baseline_cat_gender, y=MEAN, color=bmi_baseline_cat, linetype=gender))+
  geom_point(position = position_dodge(width=1))+
  scale_color_manual(values = c(
    "blue",
    "forestgreen",
    "orange"))+
  scale_linetype_manual(values=c("solid","dashed"))+
  geom_errorbar(aes(ymin=`2.5 %`, ymax=`97.5 %`),position = position_dodge(width=1))+
  theme_bw()+
  facet_wrap(~Param, scales = "free_y") +
  guides(color=guide_legend(title="BMI Category"), linetype=guide_legend(title="Gender"))+
  labs(x="BMI * Gender", y="Mean (CI 95%)") + 
  theme(strip.background = element_rect(color="black", fill="white", linetype=0))+
  theme(strip.text.x = element_text(size = 14), axis.text.x=element_text(angle=45,hjust=1))
## The code below create the object stat.test, so you can add stars to the plot and it requires db.delta
require(rstatix)
stat.test<-stat.test.stars(db.delta=db.delta, contrast.frm = "~bmi_baseline_cat|gender")
require(ggpubr)
## comparisons of interests
comparisons<-c("Normal,Woman_vs_Overweight,Woman" , "Normal,Woman_vs_Obese,Woman" , "Overweight,Woman_vs_Obese,Woman" ,
               "Normal,Man_vs_Overweight,Man"  ,     "Normal,Man_vs_Obese,Man","Overweight,Man_vs_Obese,Man",
               "Normal,Woman_vs_Normal,Man" , "Overweight,Woman_vs_Overweight,Man","Obese,Woman_vs_Obese,Man")
stat.stars<-subset(stat.test, stat.test$contrast %in% comparisons)
p<-p+stat_pvalue_manual(stat.stars, hide.ns = T, step.increase = 0.05, tip.length = 0.03, step.group.by="Param")
p
```


Step 5. plot the COSINOR circadian curves

```{r, warning=FALSE,  fig.width=10}
## This code creates blank axis
p2<-ggplot.cosinor.lmer(object=f3,
                       x_str="bmi_baseline_cat_gender",
                       period=24,
                       db.means=db.means,
                       DATA=db.model)
## This code customizes the plots
p2<-p2 + labs(x="Time (hours)", y="HRV SDNN, (ms)")+
  scale_color_manual(values=c(   
    "blue",
    "forestgreen",
    "orange",
    "red",
    "purple",
    "deepskyblue"))+
  geom_hline(aes(yintercept=MESOR), linetype=2, color="black")+
  theme_bw()+
  facet_wrap(~bmi_baseline_cat_gender, nrow = 1)+
  guides(color=guide_legend(title="BMI * Gender Category"))+
  geom_segment(aes(x = T_AMP, 
                   y = MESOR+0.05, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude-0.05),
               arrow = arrow(length = unit(0.15, "cm"),ends="both"), 
               size=0.5, color="gray50",linetype=1)+
  geom_segment(aes(x = 0, 
                   y =  MESOR+Amplitude+0.1, 
                   xend = T_AMP, 
                   yend = MESOR+Amplitude+0.1),
               arrow = arrow(length = unit(0.15, "cm")), size=0.5, color="gray50",linetype=1)+
  geom_line(aes(color=factor(GROUP, levels = c("Normal,Woman","Overweight,Woman","Obese,Woman",
                                               "Normal,Man","Overweight,Man","Obese,Man"))))+
  theme(strip.background = element_rect(color="black", fill="white", linetype=0))+
  theme(strip.text.x = element_text(size = 8))
p2
```

