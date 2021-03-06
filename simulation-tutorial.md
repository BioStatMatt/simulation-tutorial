<!-- <style> -->
<!-- blockquote { -->
<!-- background-color: #a6bddb50; -->
<!-- font-family: "helvetica"; -->
<!-- font-size:110%; -->
<!-- } -->
<!-- blockquote li{ -->
<!-- padding-bottom:10px; -->
<!-- } -->
<!-- svg { -->
<!-- width:none; -->
<!-- height:none; -->
<!-- } -->
<!-- </style> -->
How to simulate operating characteristics
=========================================

What is an operating characteristic
-----------------------------------

Operating characteristics are frequency properties of a procedure. They are the currency by which we evaluate and compare statistical procedures. Type I and II errors are opertating characteristics. So are the expected sample size, the distribution of stopping times in a clinical trial, the avergae length of a confidence interval, etc. These properties are premised on the classic "long-run" interpretation of probabilistic events. As such, they can be simulated by simply repeating the planned procedure and observing how often some event happens.

General advice for computing tasks
----------------------------------

So you want to perform a simulation study? Start with the tgs axioms of computing:

1.  **The act of turning on the computer does not magically endow you with understanding of your task.** If you do not know how you will perform an analysis or simulation before you turn on your computer, you will not know how to do it afterwards either.

2.  **Use modular/functional programming.** Functional programming means that you identify and write short, single purpose functions for each distinct task in your program. (Examples below.) This will allow you to develop your code in a systematic way, and it will provide a natural method for debugging your code. You will simply need to verify that the different sub-functions are working as expected.

The big picture of simulation studies
-------------------------------------

In a typical data analysis setting, there are population parameters that one hopes to estimate by collecting and analyzing data. The population parameters are unknown, and the accuracy of the conclusions is unknown.

![](/simulation-tutorial_files/a.svg)

In a simulation setting, the researcher sets the population parameters then generates data using the parameters. After completing the analysis, the researcher can then evaluate the accuracy of the conclusions. By repeating this process several times, the researcher can estimate the operating characteristics for *the specific set of population parameters used to simulate the data*.

**Caveat emptor:** Simulations are essentialy computational proofs on a case-by-case basis (e.g., this result happens for this parameter set, that result happens for that parameter set). It is up to you make the conneciton between different settings, notice patterns, and make a case for general patterns of behavior under certain circumstances. Simuations lack the natural generalizibiltiy of a mathematical proof, so the results don't necessary hold for all sets of parameter values.

![](/simulation-tutorial_files/b.svg)

The framework described above suggests how one may write modular code to perform the simulation. One can write a function to perform each of the primary tasks. For example:

![](/simulation-tutorial_files/c.svg)

Example A
---------

A clinical trial is designed to compare a patient's length-of-stay between a new surgical approach and the traditional surgical approach. With the traditional surgical approach, length-of-stay follows a mixture distribution. One-fourth of patients stay zero days in the hospital. The other portion have a length-of-stay distribution that follows a discretized, shifted-by-one exponential with a mean of 4 days. It is believed that the new approach will not alter the number of subjects that stay zero nights. Rather, the new approach will reduce the length-of-stay among patients that stay at least one night.

The primary endpoint (length-of-stay) between the two groups will be comapred using a Wilcoxon two-sample test.

You are asked to find the minimal detectable difference, with 80% power, if 150 patients are enrolled in each arm.

``` r
require(magrittr, quietly = TRUE)

parameters <- c(N = 150, trt_effect = 1)

generate_data <- function(parameters){
  N <- parameters[1]
  trt_effect <- parameters[2]
  trt <- rep(0:1, each = N)
  G <- rbinom(2*N, 1, p = 0.75)
  means <- 4 - 1 - trt_effect*trt
  Y_tmp <- 1 + rexp(2*N, 1/means) %>% floor
  Y <- Y_tmp*G + 0*(1-G)
  
  data.frame(Y = Y, trt = trt)
}

analysis <- function(D, alpha = 0.05){
  wilcox.test(Y~trt, data = D)$p.value < alpha
}

# This function isn't needed for this particular simulation, 
# but it might be helpful for the simulation in the qualifying exam.
oper_char <- function(x) 1*x

one_rep <- function(parameters) parameters %>% generate_data %>% analysis %>% oper_char
# Note that sim_params = c(M, parameters)
M_rep <- function(sim_params) replicate(sim_params[1], one_rep(sim_params[2:3]))
```

### Simulation settings and simulation

``` r
ss1 <- expand.grid(
    M = 1e4
  , N = 150
  , trt_effect = seq(0, 2, by = 1/3) %>% round(2)
  , power = NA
)

# Uses previous results if available
if("simulation-results.rds" %in% list.files()){
  ss1 <- readRDS("simulation-results.rds")
}else{
  for(i in 1:nrow(ss1)){ss1[i,4] <- ss1[i,1:3] %>% as.numeric %>% M_rep %>% mean}  
  saveRDS(ss1, "simulation-results.rds")
}
```

### Results

Tom -- The sample size is 300 (150 per group), no? Would be good to clarify...

``` r
ss1 %>% kable
```

|      M|    N|  trt\_effect|   power|
|------:|----:|------------:|-------:|
|  10000|  150|         0.00|  0.0524|
|  10000|  150|         0.33|  0.0751|
|  10000|  150|         0.67|  0.1838|
|  10000|  150|         1.00|  0.3935|
|  10000|  150|         1.33|  0.6565|
|  10000|  150|         1.67|  0.8970|
|  10000|  150|         2.00|  0.9846|

### Power plot

``` r
tgsify::plotstyle(style = upright)
plot(ss1$trt_effect, ss1$power, xlab = "Treatment effect", ylab = "Power")
lm1 <- glm(power ~ poly(trt_effect, degree = 3, raw = TRUE), data = ss1, family = binomial)
xx <- seq(0,2,by=.1)
power_smooth <- predict(lm1, newdata = data.frame(trt_effect = xx), type = "response")
lines(xx, power_smooth, lwd = 3)
abline(h = 0.8, col = "grey")
(power_smooth - 0.8) %>% abs %>% which.min %>% `[`(xx, .) %>% abline(v = ., col = "grey")
```

![](simulation-tutorial_files/figure-markdown_github/unnamed-chunk-7-1.svg)

Conclusion: And we can see from the plot that ...

Tom -- I think it would be helpful to connect the parameter to the shift in the \#days. Either by adding it as an axis or by just adding it to the table. This way folks will make the connection between the parameter space, the change in outcome, and how often we detect it.

Example B
---------

A pharmaceutical company has been commissioned to rapidly develop a new antibiotic that will be more effective against a bacterial infections that are highly resistant to standard therapy. To speed up development, the research team has decided to simultaneously test 10 new candidate medications against standard therapy in a randomized controlled trial. This has created methodological challenges that the biostatistical team has been asked to resolve.

The study is designed to collect the outcome (infection resolved or not resolved within 14 days) on 400-500 patients for the standard therapy (control arm) and 50-100 patients for each of the 10 candidate therapies (intervention arms). For sake of estimating the operational characteristics of the proposed analysis approaches, they agree to assume that the sample sizes are uniformly distributed between 50 and 100, or 400 and 500, as the case may be. They will also assume that all patients' outcomes are independent of each other and the probability of resolution under standard therapy is a measily 0.10.

A group of biostatisticians (Group One) is concerned about controlling the familywise Type I error (FWER) when testing the 10 treatments against the control group all at once. They propose the following: first perform a single chi-square test on all 11 groups to test for any differences. This test would be evaluated at a 5% significance level. If it was not significant, they would conclude there were no differences in resolution rates. If it was significant, they would then individually test each of the treatments against the control with a chi-square test each at a 5% significance level. Any of these tests that return a p-value &lt; 0.05 would be deemed statistically significant.

Another group of biostatisticians (Group Two) is concerned that controlling the FWER could reduce the power too much. They propose going straight to individually testing each of the treatments against the control with chi-square tests, but using a Bonferroni correction to maintain a 5% familywise error rate. Any of these tests that return p-values less than the Bonferroni threshold would be deemed statistically significant.

A third group of biostatisticians (Group Three) is concerned about ignoring what is clinically meaningful or meaningless. They ask the researchers what a clinically meaningful improvement would be. The researchers reply around a 15% absolute gain, e.g. if the control had a 10% resolution rate, a treatment with a 25% rate would be truly meaningful. Then the biostatisticians ask what would be a clinically trivial change from the standard therapy. The team replies anything within a 2% absolute difference, e.g. going from 10% to 12%, would be trivial -- essentially meaningless. Group THREE proposes calculating the 95% confidence intervals for the risk difference for each of the 10 interventions vs the control, and deeming any interval that falls above a 2% absolute improvement to be clinically nontrivial, i.e. when the CI has a lower bound &gt; 0.02.

For the tests, all the biostatisticians agree on using the default chi-square test function in R, chisq.test(), with the default function settings. For the risk difference confidence intervals, they agree to use the Agresti-Caffo interval as implemented by the wald2ci() function in the R package PropCIs.

1.  Write a brief paragraph comparing the familywise Type I error rates (FWER) of the three analysis methods (Proposals ONE, TWO, and THREE) for testing the 10 new therapies versus the control therapy. Report your error rates to three decimal places and design your methods so you have a high degree of certainty on the first two decimal places. <br><br> Write a second paragraph describing your methods for estimating these error rates in detail and provide supporting code and figures as appropriate. – Note: Clearly explaining what you were trying to do is very important if it turns out you have an error in your code. It helps the graders give you partial credit.
    <blockquote>
    <ul>
    <li>
    Write the functions <tt>generate\_data()</tt>, <tt>analysis1()</tt>, <tt>analysis2()</tt>, <tt>analysis3()</tt>, <tt>fwer()</tt>.
    <li>
    Write <tt>generate\_data()</tt> so that one of the inputs is a vector of event probabilities corresponding to each arm of the study.
    <li>
    Write the analysis functions in a way that the input and output is exactly the same.
    <li>
    Write <tt>fwer()</tt> in a way so that it calculates the fwer regardless of the analysis method.
    <li>
    Write a <tt>one\_rep()</tt> function so that each generated dataset is analyzed by all three methods. Perhaps the output is a triple.
    </ul>
    </blockquote>
2.  Write a brief paragraph offering intuition on why each the three methods behaved the way they did in part a. Attempt to explain why each method either 1) achieved a FWER of 5% exactly, 2) was conservative by achieving a FWER less than 5%, or 3) failed to keep FWER under 5%.

3.  Write a brief paragraph comparing the power of the three analysis methods (Groups One, Two, and Three) for detecting that one new treatment, call it treatment A, is different than the control therapy assuming treatment A has a resolution rate of 26% and all the other therapies retain the control rate of 10%. Here power is referring to the probability of detecting the effect of treatment A specifically. Remember: Missing the effect of A and finding a false positive effect in a different therapy doesn't count as a successful study. Report your Power estimates to three decimal places and design your methods so you have a high degree of certainty on the first two decimal places.<br> <br> Write a second paragraph describing your methods for estimating these error rates. Be specific, provide detials, and attach supporting code and figures as appropriate.
    <blockquote>
    You should be able to complete this step by writing 1 additional function and reusing the code from part (a).
    </blockquote>
4.  Write a brief paragraph commenting on which method had the best Power out of the methods that controlled FWER at 5% or less. Offer insight into why the best performing method outperformed the others.

5.  Assume that the settings for parts a and c were the only two scenarios that could occur with non-negligible probability and that they were equally likely to happen. Write a brief paragraph commenting on the False Discovery and False Non-Discovery Rates of the three approaches.<br> <br> Write a second paragraph describing your methods for estimating the FDR and FNR in detail and provide supporting code and figures as appropriate. After the study was done and it was being analyzed, analysts wouldn’t omnisciently know there is at most one real effect like we are assuming here. So it is important to allow the methods to potentially find more than one significant result in a given study.<br>
    <blockquote>
    You'll need to write an additional function for this step. It will also require you two output more than a single number from the <tt>oper\_char()</tt> function.
    </blockquote>
6.  Explain why the FDR and FNR are of interest to the statisticians designing the study. Attempt to offer insight into why the best performing method outperformed the others.
