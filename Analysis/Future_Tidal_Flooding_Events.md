How Will SLR Increase Risk of Tidal Flooding?
================
Curtis C. Bohlen
December 16, 2020

-   [Introduction](#introduction)
-   [Import Libraries](#import-libraries)
-   [Import Data](#import-data)
    -   [Combine Data](#combine-data)
-   [Reprise the MGS Analysis](#reprise-the-mgs-analysis)
    -   [Cleanup](#cleanup)
-   [Evaluating Data for Simulations](#evaluating-data-for-simulations)
    -   [Long Term trend](#long-term-trend)
        -   [Focus on The Official Tidal
            Epoch](#focus-on-the-official-tidal-epoch)
    -   [Distribution of Deviations](#distribution-of-deviations)
        -   [Calculate the Moments of the
            Distribution](#calculate-the-moments-of-the-distribution)
        -   [Conclusion](#conclusion)
    -   [Temporal Autocorrelation](#temporal-autocorrelation)
        -   [Conclusions](#conclusions)
-   [Simulation Models](#simulation-models)
    -   [Resampling Model](#resampling-model)
        -   [Evaluating Resampled
            Deviations](#evaluating-resampled-deviations)
        -   [Resampling Function](#resampling-function)
        -   [Test Our Function](#test-our-function)
        -   [Run Full Simulation](#run-full-simulation)
    -   [ARIMA Models](#arima-models)
        -   [Developing an ARIMA
            Simulator](#developing-an-arima-simulator)
        -   [Evaluating ARIMA
            Simulations](#evaluating-arima-simulations)
        -   [Simulation Function](#simulation-function)
        -   [Test Our Function](#test-our-function-1)
        -   [Run Full Simulation](#run-full-simulation-1)
-   [Wrapping it all up](#wrapping-it-all-up)
    -   [Direct Computation](#direct-computation)
    -   [Resampling](#resampling)
    -   [ARIMA](#arima)
    -   [Make a Nice Table](#make-a-nice-table)
-   [Updated To Present Conditions](#updated-to-present-conditions)
    -   [Updated Direct Computation](#updated-direct-computation)
    -   [Updated Simulation](#updated-simulation)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

One reviewer of our draft Climate Change chapter pointed out that for
all weather-related events, we presented data on the changing frequency
of “events” over time, including hot days, cold days, large storms, etc.
They suggested we show similar graphics showing changes in frequency of
tidal flooding events. We prepared that graphic in
[Tidal\_Flooding\_Events.Rmd](Graphics/Tidal_Flooding_Events.Rmd).

In this notebook, we to put that graphic in context by looking at
estimates of future tidal flooding risk under a few SLR scenarios. Our
goal is to be able to say that analysis or simulation suggests an X%
increase in frequency of flooding with a Y foot increase in SLR.

In our analysis, we follow Maine Geological Survey’s practice of
declaring a tidal flooding event whenever tidal observations exceed the
current “Highest Astronomical Tide” or HAT level, which is 11.95 feet,
or 3.640 meters, above mean lower low water (MLLW) at Portland.

Maine Geological Survey has estimated future flooding risk by adding a
SLR value (0.8 foot and 1.5 feet) to the historical record, and showing
how frequent flooding would have been under SLR conditions. This
provides a ready estimate of impact of SLR on frequency of flooding.

Details of their method and a data viewer for different locations in
Maine are available at the [Maine Geological Survey sea level rise data
viewer](https://mgs-collect.site/slr_ticker/slr_dashboard.html).

We (nearly) repeat their analysis for selected SLR scenarios ourselves,
over a fixed period of time (the past 20 years) and estimate percentage
change in flooding under SLR. Our results will differ from theirs, as we
count days per year with flooding, while they count hours.

We then go on to simulate flooding histories based on the 19 year tidal
epoch, and examining predicted flood frequencies using three different
models under one foot, two foot, and three foot SLR scenarios.

Many of the ideas first developed in this Notebook have since been
incorporated into the `SLRSIM` package. The goal of that package is so
simplify analysis of historic SLR. A draft version of the package is
available at [SLRSIM](https://github.com/ccb60/SLRSIM).

# Import Libraries

``` r
library(tidyverse)
#> Warning: package 'tidyverse' was built under R version 4.0.5
#> -- Attaching packages --------------------------------------- tidyverse 1.3.1 --
#> v ggplot2 3.3.5     v purrr   0.3.4
#> v tibble  3.1.6     v dplyr   1.0.7
#> v tidyr   1.1.4     v stringr 1.4.0
#> v readr   2.1.0     v forcats 0.5.1
#> Warning: package 'ggplot2' was built under R version 4.0.5
#> Warning: package 'tidyr' was built under R version 4.0.5
#> Warning: package 'dplyr' was built under R version 4.0.5
#> Warning: package 'forcats' was built under R version 4.0.5
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
library(readr)

library(moments)  # for skewness and kurtosis; we could calculate, but why?
library(forecast) #for auto.arima()
#> Registered S3 method overwritten by 'quantmod':
#>   method            from
#>   as.zoo.data.frame zoo

library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())
```

# Import Data

Our primary source data is hourly data on observed and predicted water
levels at the Portland tide station (Station 8418150). We accessed these
data using small python scripts to download and assemble consistent data
from the NOAA Tides and Currents API. Details are provided in the
“Original\_Data” folder.

``` r
sibfldnm <- 'Data'
parent <- dirname(getwd())
sibling <- file.path(parent,sibfldnm)

#dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
```

``` r
fn <- 'portland_tides_hourly.csv'
fpath <- file.path(sibling, fn)

observed_data  <- read_csv(fpath, col_types = cols(Time = col_time('%H:%M'))) %>%
  rename(MLLW = `Water Level`,
         theDate =`Date`) %>%
  mutate(Year = as.numeric(format(theDate, '%Y')),
         MLLW_ft = MLLW * 3.28084,
         Exceeds = MLLW > 3.640)
```

``` r
fn <- 'portland_tides_hourly_predicts.csv'
fpath <- file.path(sibling, fn)

predict_data  <- read_csv(fpath, col_types = cols(Time = col_time('%H:%M'))) %>%
  rename(theDate =`Date`) %>%
  mutate(Hour  = as.numeric(format(DateTime, '%H')),
         Month = as.numeric(format(theDate, '%m')),
         Day   = as.numeric(format(theDate, '%d')),
         Year  = as.numeric(format(theDate, '%Y'))) %>%
  select(-DateTime, -theDate, -Time)
```

## Combine Data

The number of predicted and observed values are not **quite** the same,
so we need to make sure we merge data appropriately. We merge by Year,
Month, Day, and Hour. We could use DateTime, but we have not checked if
both files use the same conventions throughout. This is likely more
robust.

``` r
combined <- observed_data %>%
  mutate(Hour  = as.numeric(format(DateTime, '%H')),
         Month = as.numeric(format(theDate, '%m')),
         Day   = as.numeric(format(theDate, '%d')),
         Year  = as.numeric(format(theDate, '%Y'))) %>%
  select(-Sigma, -MLLW_ft, -Exceeds) %>%
  inner_join(predict_data,by = c("Year", "Hour", "Month", "Day"))

## Calculate Deviations Between Predicted and Observed
combined <- combined %>%
  mutate(deviation = MLLW - Prediction)
```

# Reprise the MGS Analysis

We reprise the analysis conducted by Maine Geological Survey. MGS added
fixed estimates of SLR to the historic record, and counted hours of time
with coastal flooding.

We take the same approach to identify hours with coastal flooding under
one foot, two foot, and three foot SLR scenarios (MGS used 0.8 and 1.5
foot scenarios), but then count **days** in which flooding occurs. We
focus on impact of moderate SLR on coastal flooding compared to the
frequency of flooding observed over the past 20 years.

We use the Tidyverse for these relatively simple calculations, relying
on methods developed to produce graphics showing number of days with
flooding over the period of record. (See [Tidal Flooding
Events.Rmd](Graphics/Tidal%20Flooding%20Events.Rmd).

``` r
HAT <- 11.95
slr_annual <- observed_data %>%
  filter(Year > 2000) %>%
  mutate(slr_1 = MLLW_ft + 1,
         slr_2 = MLLW_ft + 2,
         slr_3 = MLLW_ft + 3,
         exceeds_0 = MLLW_ft > HAT, 
         exceeds_1 = slr_1   > HAT,
         exceeds_2 = slr_2   > HAT,
         exceeds_3 = slr_3   > HAT) %>%
  group_by(theDate) %>%
  summarize(Year = first(Year),
            exceeded_0 = any(exceeds_0, na.rm = TRUE),
            exceeded_1 = any(exceeds_1, na.rm = TRUE),
            exceeded_2 = any(exceeds_2, na.rm = TRUE),
            exceeded_3 = any(exceeds_3, na.rm = TRUE),
            n = sum(! is.na(Exceeds)),
            .groups = 'drop') %>%
  group_by(Year) %>%
  summarize(Days = n(),
            floods_0 = sum(exceeded_0),
            floods_1 = sum(exceeded_1),
            floods_2 = sum(exceeded_2),
            floods_3 = sum(exceeded_3),
            .groups = 'drop')
```

``` r
summ <- slr_annual%>%
  filter(Year> 1999) %>%
  summarize(across(contains('floods'), mean, na.rm = TRUE))
summ
#> # A tibble: 1 x 4
#>   floods_0 floods_1 floods_2 floods_3
#>      <dbl>    <dbl>    <dbl>    <dbl>
#> 1     7.37     62.6     195.     330.
```

That suggests a change in flooding frequency on the order of 8.5 with
one foot of SLR.

## Cleanup

``` r
rm(summ, observed_data, predict_data)
```

# Evaluating Data for Simulations

We continue, working with data where tide levels are expressed in meters
above MLW. Here we evaluate the data in terms of how best to simulate
future flooding events.

## Long Term trend

``` r
ggplot(combined, aes(DateTime, deviation)) +
  geom_point(size = 0.01, alpha = 0.05, color = 'yellow') +
  geom_smooth(method = 'lm')
#> `geom_smooth()` using formula 'y ~ x'
#> Warning: Removed 22718 rows containing non-finite values (stat_smooth).
#> Warning: Removed 22718 rows containing missing values (geom_point).
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/trend_in_deviations_plot-1.png" style="display: block; margin: auto;" />

``` r
test <- lm(deviation ~ theDate, data = combined)
coef(test)
#>   (Intercept)       theDate 
#> -2.893013e-02  5.138791e-06
```

We do not show error estimates of this regression, as observations are
highly autocorrelated, and estimated error would be misleading, but the
slope is meaningful. The implied rate of annual increase in deviations
is

``` r
unname(round(coef(test)[2]*1000*365.25,2))
#> [1] 1.88
```

Here, the regression slope suggests a 1.88 mm annual rate of increase in
the deviations between predicted and observed water levels. Not
coincidentally, that value is remarkably close to the long-term estimate
of SLR at the Portland station.

### Focus on The Official Tidal Epoch

Tidal predictions are defined in terms of a specific 19 year long “Tidal
Epoch.” The astronomical alignments of sun, moon, and earth repeat (at
least closely enough for tidal prediction) every nineteen years, and
tides are predicted based on astronomical processes. Tidal “Predictions”
are based on a complex periodic function that is parameterized by
“harmonic constituents”, themselves calculated based on observed tidal
elevations over the 19 year Tidal Epoch.

The current tidal epoch for Portland is 1983-2001. (According to the
NOAA web page for the station. The epoch is provided on the datums
sub-page.)

For our purposes, the key insight is that tidal predictions are
effectively a nineteen year-long periodic function, and thus do not take
into account changes in sea level. Consequently, the deviations from
tidal predictions we calculated are not stationary.

However, during the Tidal Epoch, the average error of prediction should
be zero (or very close to zero). Our best understanding of the
distribution of deviations from (astronomical) predicted tide
elevations, therefore, would come from looking at deviations during that
tidal epoch, when deviations due to changing sea level are minimized.

``` r
epoch <- combined %>%
  filter(Year> 1982 & Year < 2002)
```

``` r
test <- lm(deviation ~ theDate, data = epoch)
coef(test)
#>   (Intercept)       theDate 
#> -1.610238e-02  1.904924e-06
unname(round(coef(test)[2]*1000*365.25,2))
#> [1] 0.7
```

We do not show error estimates of this regression either, but it
suggests a change in deviations one third as great as observed over the
entire period of record, at less than a mm a year.

## Distribution of Deviations

We need to figure out how best to model future deviations between
observed and predicted tide levels. In particular, we need to identify
strategies for creating a reasonable “random” version of deviations so
that we can simulate future conditions.

**Values of Tide Levels in the rest of the Notebook are in meters.**

``` r
ggplot() +
  geom_histogram(aes(x = deviation, y = ..density..),
                 bins = 100, data = epoch) +
  geom_density(aes(x = deviation), color = 'red', data = epoch) +
  stat_function(fun = dnorm,
                args = list(mean = mean(epoch$deviation, na.rm = TRUE),
                                   sd = sd(epoch$deviation, na.rm = TRUE)),
                color = 'blue') +
  geom_vline (xintercept = 0, color = 'red')
#> Warning: Removed 1384 rows containing non-finite values (stat_bin).
#> Warning: Removed 1384 rows containing non-finite values (stat_density).
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/hist_deviations-1.png" style="display: block; margin: auto;" />

So, deviations are “close” to normally distributed, or at least bell
shaped, with mean close to zero. A mean close to zero is not due to
chance, but the nearly normal distribution reflects something about the
nature of atmospheric and other non-astronomical effects on local sea
level.

### Calculate the Moments of the Distribution

``` r
mean(epoch$deviation, na.rm = TRUE)
#> [1] -0.0004384899
sd(epoch$deviation, na.rm = TRUE)
#> [1] 0.1283451
skewness(epoch$deviation, na.rm=TRUE)
#> [1] 0.4148183
kurtosis(epoch$deviation, na.rm = TRUE)
#> [1] 5.562658
```

Formal statistical tests will do us little good here. With this much
data, nothing ever matches theoretical distributions exactly. This is a
matter for judgment, not testing.

#### Skewness

A non-zero skew means the distribution is not symmetrical about its
mean. A positive skew suggests a few extra observations below the mean,
and more extreme observations above the mean. The median is lower than
the mean. The upper tail is longer than the lower tail.

We can confirm that somewhat more observations are below then mean than
above.

``` r
below <- sum(epoch$deviation < mean(epoch$deviation, na.rm = TRUE),
             na.rm = TRUE)
above <- sum(epoch$deviation > mean(epoch$deviation, na.rm = TRUE),
             na.rm = TRUE)
below/(below+above)
#> [1] 0.5190464
```

So we have a couple of percent more points below the mean than a normal
distribution. We must have a few extreme values above the mean to
balance the distribution about the mean.

#### Kurtosis

Positive kurtosis suggests a “fat tailed” distribution. That is, there
are more observations in the center and tails of the distribution than
would be true of a normal distribution with similar standard deviation.
Again, we have a few more extreme values.

### Conclusion

We are interested in the extremes of this distribution – or the
frequency of extremes. The effects of skewness and kurtosis on frequency
of extreme values reinforce each other. There will be more extreme
values than under a normal distribution.

Simulation with a normal distribution would probably be a mistake. We
need to either use a distribution where we can tune the skewness and
kurtosis, resample from the observed distribution of deviations, or use
more sophisticated tools.

## Temporal Autocorrelation

``` r
acf(epoch$deviation, na.action = na.pass, lag.max = 24*5)
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/acf_3-1.png" style="display: block; margin: auto;" />

So, we see high autocorrelation, with a roughly 12 hour (tidal) cycle
imposed over a gradual decay. Autocorrelation remains present over a
period of days. It’s not obvious whether that 12 hour pattern is
physical or a mathematical artifact of how the predicted tides are
calculated.

``` r
pacf(epoch$deviation, na.action = na.pass)
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/pacf_3-1.png" style="display: block; margin: auto;" />
Partial autocorrelations at lag 1 are high, so the core of any ARIMA
model must be a lag 1 term, probably an AR(1) process. There is also
some quasi-periodicity at weird 7 hour and 14 hour intervals, which
suggests we may need some other terms.

### Conclusions

We can’t assume a simulation that does not address autocorrelation will
be adequate.

# Simulation Models

We develop two simulation models that estimate future flooding by adding
simulated deviations to predicted tides.

1.  Start with predicted tides from the official tidal epoch (in
    meters)  
2.  Add (or not) a sea level rise value to the predicted tides  
3.  Add a random deviation from predicted tidal elevation, where the
    random deviation is derived from either resampling from hourly
    deviations or via simulating an ARIMA process.  
4.  Count up the number of days with simulated flood events over the
    tidal epoch, and divide by 365.25 to estimate floods per year.

In each Model, simulate 1000 mock tidal epochs, and look at properties
of the resulting distribution of estimated rates of flooding per year.

## Resampling Model

The first model resamples deviations from the historically observed
deviations from predicted tides.

### Evaluating Resampled Deviations

We can simulate a “random” time series by resampling from observed
values of the deviation between predicted and observed tidal levels.

``` r
tmp <-  epoch$deviation[! is.na(epoch$deviation)] # don't sample NAs
test <- sample(tmp, length(epoch$deviation), replace = TRUE)
```

``` r
ggplot() + 
  geom_histogram(aes(x = test, y = ..density..), bins = 100) +
  geom_density(aes(x = test), color = 'red') +
  stat_function(fun = dnorm, args = list(mean = mean(test), sd = sd(test)),
                color = 'blue') +
    geom_vline (xintercept = 0, color = 'red') 
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/hist_test_resample-1.png" style="display: block; margin: auto;" />

Lets look at the moments of the simulated data. (Results will vary; this
is a simulation).

``` r
mean(test)
#> [1] -0.0003979287
sd(test)
#> [1] 0.1284654
skewness(test)
#> [1] 0.4139797
kurtosis(test)
#> [1] 5.556434
```

Moments are remarkably similar to source data, as would be expected.

``` r
ggplot() +

  geom_line(aes(x = (1:(24*50)), y = test[1:(24*50)]), alphs = 0.75) +
  geom_line(aes(x = (1:(24*50)), y = epoch$deviation[1:(24*50)]),
            color = 'orange', alpha = 0.75) +
   
  scale_x_continuous(breaks = seq(0, 24*100, 24*20)) +
  ylab('Deviances (m)') +
  xlab('Hours') +
  labs(subtitle = 'Black is Simulated, Orange is Observed')
#> Warning: Ignoring unknown parameters: alphs
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/lines_test_resample-1.png" style="display: block; margin: auto;" />
There is a **lot** more temporal structure in the original data than in
the simulated data. This is because the original data was
autocorrelated, and simple random (re)sampling eliminates the temporal
autocorrelation.

### Resampling Function

This is the workhorse function that carries out the steps just described
(except the repeat sampling).

``` r
resample_once <- function(dat, pr_sl, dev, dts, slr) {
  # data is a dataframe
  # pr_sl is the data column containing the (astronomical) sea level predictions
  # dev is the data column containing observed deviations from predictions
  # dts is a data column of identifiers by date / day
  # slr is the value for SLR for estimating future tidal elevations 
  
  # Returns the mean number of floods per year over the simulated tidal epoch.
  
  
  # We quote data variables, and look them up by name
  # Caution:  there is no error checking.
  pr_sl <- as.character(ensym(pr_sl))
  dev <- as.character(ensym(dev))
  dts <- as.character(ensym(dts))
  
  tmp <-  dat[[dev]]
  tmp <-  tmp[! is.na(tmp)]
  #Simulate one tidal epoch of hourly tidal elevations
  val <- dat[[pr_sl]] + slr + sample(tmp,
                                     length(dat[[pr_sl]]),
                                     replace = TRUE)
  
  #create a dataframe, for convenient calculations
  df <- tibble(theDate = dat[[dts]], sim = val)
  
  #Calculate results (DAYS with tidal elevations above HAT == 3.64 m)
  res <- df %>%
    group_by(theDate)  %>%
    summarize(exceeded = any(sim > 3.640),
              .groups = 'drop') %>%
    summarize(days = sum(! is.na(exceeded)),
              floods = sum(exceeded, na.rm = TRUE),
              floods_p_yr = 365.25 * floods/days) %>%
    pull(floods_p_yr)
  
  return(res)
}
```

### Test Our Function

``` r
resample_once(epoch, Prediction, deviation, theDate, 0)
#> [1] 5.420857
```

### Run Full Simulation

the following takes about thirty seconds to run.

``` r
set.seed(12345)
samp = 1000
res <- numeric(samp)
for (iter in seq(1, samp)) {
  res[[iter]] <- resample_once(epoch, Prediction, deviation, theDate, 0)
}
```

#### Evaluate Results

``` r
mean(res)
#> [1] 6.038519
sd(res)
#> [1] 0.5149271
skewness(res)
#> [1] -0.1970307
kurtosis(res)
#> [1] 2.973823
```

``` r
ggplot() +
  geom_histogram(aes(x = res, y = ..density..)) +
  geom_density(aes(x = res), color = 'red') +
  stat_function(fun = dnorm, args = list(mean = mean(res), sd = sd(res)),
                color = 'blue')
#> `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/hist_resample_results-1.png" style="display: block; margin: auto;" />

## ARIMA Models

The prior analysis did not take into account the temporal
autocorrelation of deviations from predicted tidal heights. In this
section, we take a similar simulation approach, but simulate a slightly
more faithful representation of the distribution of deviations from
predicted tidal heights, by simulating an ARIMA process.

### Developing an ARIMA Simulator

We can fit an ARIMA model to the tidal deviations data, and then
simulate random deviations based on the parameters we find. We fit an
ARIMA model using the `auto.arima()` function from the `forcast`
package.

By definition, the observed deviations are stationary, with mean zero,
so we need not do any preliminary differencing. We signal that to
`auto.arima()` with d = 0. We signal a stationary, non-seasonal process,
with zero mean as well, to take advantage of properties of the
deviations.

Although our time series has a clear “seasonal” structure at roughly a
12 or 13 hour period (half of 25 hour, which is close to the tidal cycle
of 24 hours 50 minutes), seasonal ARIMAs are very slow to fit. Even
first order seasonal terms take four or five minutes to fit. Second
order and higher terms take substantially longer. An exhaustive search
bogged down immediately.

We first search for non-seasonal ARIMA models, and then refit similar
seasonal models. (We explored this problem in greater depth when
developing our `SLRSIM` package, but we did not develop those ideas in
time for inclusion in State of Casco Bay.)

We speed things up by setting `stepwise` and `approximation` arguments
to `FALSE`. You can request a trace to show how the model search
progressed, as we have done. You can also decide whether to fit seasonal
terms, but to do that, you have to pass a frequency to the time series
object.

We get a qualitatively different fifth order ARIMA model if we set
`stepwise` to `FALSE`. The alternate model (not shown) has one AR term
and four MA terms.

This takes around 20 seconds to run.

``` r
my_arima <- auto.arima(epoch$deviation,
                       d = 0,                # No differencing.
                       stationary = TRUE,
                       seasonal = FALSE,
                       allowdrift = FALSE,   # not sure how this differs from "stationary"
                       allowmean = FALSE,    # Our time series has mean = 0
                       stepwise = TRUE,      # FALSE is faster, less accurate
                       approximation = TRUE, # FALSE is slower
                       trace = TRUE)
#> 
#>  Fitting models using approximations to speed things up...
#> 
#>  ARIMA(2,0,2) with zero mean     : -630977
#>  ARIMA(0,0,0) with zero mean     : -209471.2
#>  ARIMA(1,0,0) with zero mean     : -624631.3
#>  ARIMA(0,0,1) with zero mean     : -398718.4
#>  ARIMA(1,0,2) with zero mean     : -627271.8
#>  ARIMA(2,0,1) with zero mean     : -630074
#>  ARIMA(3,0,2) with zero mean     : -634102.4
#>  ARIMA(3,0,1) with zero mean     : -631709.1
#>  ARIMA(4,0,2) with zero mean     : -638134.9
#>  ARIMA(4,0,1) with zero mean     : -638119.7
#>  ARIMA(5,0,2) with zero mean     : -638140.5
#>  ARIMA(5,0,1) with zero mean     : -638140.8
#>  ARIMA(5,0,0) with zero mean     : -640629.6
#>  ARIMA(4,0,0) with zero mean     : -639491.8
#> 
#>  Now re-fitting the best model(s) without approximations...
#> 
#>  ARIMA(5,0,0) with zero mean     : -640613.6
#> 
#>  Best model: ARIMA(5,0,0) with zero mean
#my_arima
(my_coefs <- coef(my_arima))
#>         ar1         ar2         ar3         ar4         ar5 
#>  1.17564061 -0.26121524  0.24867580 -0.30207238  0.08292214
(my_sigma2 <- my_arima$sigma2)
#> [1] 0.001210982
```

``` r
acf(my_arima$residuals,  na.action = na.pass, lag.max = 24*5)
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/acf_arima_residuals-1.png" style="display: block; margin: auto;" />

``` r
pacf(my_arima$residuals,  na.action = na.pass, lag.max = 24*5)
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/acf_arima_residuals-2.png" style="display: block; margin: auto;" />
so we have gotten rid of most of the non-periodic structure, and the
structure under a period of a few hours, but residuals still show
higher-order periodic structure.

A “Seasonal” ARIMA that includes a 25 hour quasi-tidal period reduces,
but does not eliminate the periodic components in the ACF and PACF.
Magnitudes are reduced by a third to a half. But it takes a long time to
run, and base R and `forcast` have no suitable functions for simulating
seasonal ARIMA models, although some other packages apparently do. We do
not continue with exploration of seasonal ARIMA models here.

### Evaluating ARIMA Simulations

We can simulate a “random” time series that matches the estimated ARIMA
structure of the observed deviations using `arima.sim()`.

``` r
test <- arima.sim(n = 365*24,
                  model = list(ar = my_coefs),
                  sd = sqrt(my_sigma2))
```

``` r
ggplot() +
  geom_histogram(aes(x = test, y = ..density..), bins = 100) +
  geom_density(aes(x = test), color = 'red') +
  stat_function(fun = dnorm, args = list(mean = mean(test), sd = sd(test)),
                color = 'blue') +
    geom_vline (xintercept = 0, color = 'red') 
#> Don't know how to automatically pick scale for object of type ts. Defaulting to continuous.
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/histogram_arima_sim-1.png" style="display: block; margin: auto;" />

The range of simulations is perhaps a little narrower than the observed
data, but since we simulated only 1/20 the data, that is not surprising.

Lets look at the moments of the simulated data. (Results will vary; this
is a simulation).

``` r
mean(test)
#> [1] 0.006795336
sd(test)
#> [1] 0.132654
skewness(test)
#> [1] 0.03278123
kurtosis(test)
#> [1] 2.921464
```

Mean and SD are OK fits. Both skewness and kurtosis are lower than the
historical data, but not too far off.

We also examine temporal patterns and autocorrelation.

``` r
ggplot() +
  geom_line(aes(x = (1:(24*100)), y = epoch$deviation[1:(24*100)]),
            color = 'orange', alpha = 0.5) +
  geom_line(aes(x = (1:(24*100)), y = test[1:(24*100)])) +
   
  scale_x_continuous(breaks = seq(0, 24*100, 24*20)) +
  ylab('Deviances (m)') +
  xlab('Hours') +
  labs(subtitle = 'Black is Simulated, Orange is Observed')
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/lines_test_arima_sim-1.png" style="display: block; margin: auto;" />

``` r
acf(test, na.action = na.pass, lag.max = 24*5)
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/acf_arima_sim-1.png" style="display: block; margin: auto;" />

The overall temporal pattern of the simulation looks roughly comparable.
The autocorrelation structure shows a more rapid decrease in
correlation, and a negative correlation pattern out about three days.
The modeled values lack the prominent periodic components of the real
data. There is somewhat more long-term structure in the observed data,
but we’re not far off.

### Simulation Function

We create a function that simulates a possible time series of tidal
heights. This works in direct analogy to the resampling function,
`resample_once()`. We replace the resampling mechanism for generating a
future stream of deviations from tidal predictions used there with one
based on simulating an ARIMA process.

Note that this function relies on existence of parameters and standard
error from an existing ARIMA model. We pass those as parameters,

``` r
sim_once <- function(dat, pr_sl, dts, slr,
                     coefs = my_coefs, 
                     sigma2 = my_sigma2) {
  
  # data is a dataframe
  # pr_sl is the data column containing the (astronomical) sea level predictions
  # dts is a data column of identifiers by date / day
  # slr is the value for SLR for estimating future tidal elevations 
  # coefs is a list of coefficients as produced by arima() or auto.arima()
  # sigma2 is the sigma2 measure of variation from arima() or  auto.arima()
  
  # Returns the mean number of floods per year over ONE simulated tidal epoch.
  
  # We quote data variables, and look them up by name
  # Caution:  there is no error checking.
  
  pr_sl <- as.character(ensym(pr_sl))
  #dev <- as.character(ensym(dev))
  dts <- as.character(ensym(dts))
  
  #Simulate one tidal epoch of hourly tidal elevations
  val <- dat[[pr_sl]] + slr + arima.sim(n = length(dat[[pr_sl]]),
                                 model = list(ar = coefs),
                                 sd = sqrt(sigma2))
  
  #create a dataframe, for convenient calculations
  df <- tibble(theDate = dat[[dts]], sim = val)
  
  #Calculate results
  res <- df %>%
    group_by(theDate)  %>%
    summarize(exceeded = any(sim > 3.640),
              .groups = 'drop') %>%
    summarize(days = sum(! is.na(exceeded)),
              floods = sum(exceeded, na.rm = TRUE),
              floods_p_yr = 365.25 * floods/days) %>%
    pull(floods_p_yr)
  
  return(res)
} 
```

### Test Our Function

``` r
sim_once(epoch, Prediction, theDate, 0)
#> [1] 4.420893
```

### Run Full Simulation

The following takes several minutes to run.

``` r
set.seed(54321)
samp = 1000
res <- numeric(samp)
for (iter in seq(1, samp)) {
  res[[iter]] <- sim_once(epoch, Prediction, theDate, 0)
}
```

#### Evaluate Results

``` r
mean(res)
#> [1] 4.297687
sd(res)
#> [1] 0.4760195
skewness(res)
#> [1] 0.1195593
kurtosis(res)
#> [1] 2.914246
```

Estimated number of flood days per year is slightly lower, and actually
in better agreement with the observed number during the official tidal
epoch.

Results are close to normally distributed.

``` r
ggplot() +
  geom_histogram(aes(x = res, y = ..density..), bins = 50) +
  geom_density(aes(x = res), color = 'red') +
  stat_function(fun = dnorm, args = list(mean = mean(res), sd = sd(res)),
                color = 'blue')
```

<img src="Future_Tidal_Flooding_Events_files/figure-gfm/sim_once_demo_hist-1.png" style="display: block; margin: auto;" />

# Wrapping it all up

How much will SLR increase the frequency of days with flooding? We
compare three methods. In all cases, we focus on the period of the tidal
epoch, when difference between observed and predicted tides were
minimized. That makes these results only qualitatively related to change
in frequency of flooding compared to recent times.

## Direct Computation

Following the MGS method, but focused on the same period as our
simulations for comparison purposes. Note these results are VERY
different from results for the last ten years, and in fact make the
increase in flooding frequency look much more severe.

Note we are working in meters here.

``` r
HAT <- 3.640
slr_annual <- epoch %>%
  mutate(slr_1 = MLLW + 1 * 0.3048,
         slr_2 = MLLW + 2 * 0.3048,
         slr_3 = MLLW + 3 * 0.3048,
         exceeds_0 =  MLLW   > HAT, 
         exceeds_1 = slr_1   > HAT,
         exceeds_2 = slr_2   > HAT,
         exceeds_3 = slr_3   > HAT) %>%
  group_by(theDate) %>%
  summarize(exceeded_0 = any(exceeds_0, na.rm = TRUE),
            exceeded_1 = any(exceeds_1, na.rm = TRUE),
            exceeded_2 = any(exceeds_2, na.rm = TRUE),
            exceeded_3 = any(exceeds_3, na.rm = TRUE),
            .groups = 'drop') %>%
  
  summarize(days = sum(! is.na(exceeded_0)),
            `No SLR` = 365.25 * sum(exceeded_0)/days,
            `One Foot SLR` = 365.25 * sum(exceeded_1)/days,
            `Two Foot SLR` = 365.25 * sum(exceeded_2)/days,
            `Three Foot SLR` = 365.25 * sum(exceeded_3)/days,
            `One Foot` = round(`One Foot SLR` / `No SLR`, 2),
            `Two Foot` = round(`Two Foot SLR` / `No SLR`, 2),
            `Three Foot` = round(`Three Foot SLR` / `No SLR`, 2),
            .groups = 'drop') %>%
  select(-days)
slr_annual
#> # A tibble: 1 x 7
#>   `No SLR` `One Foot SLR` `Two Foot SLR` `Three Foot SLR` `One Foot` `Two Foot`
#>      <dbl>          <dbl>          <dbl>            <dbl>      <dbl>      <dbl>
#> 1     2.11           45.8           161.             311.       21.8       76.6
#> # ... with 1 more variable: Three Foot <dbl>
v <- slr_annual$`One Foot`
```

Under that simple model, one foot of SLR increases flooding by a
whopping 21.8. Interestingly, that is mostly because the denominator –
the number of floods predicted under no sea level rise – is so small. We
use HAT as our definition of present-day flooding, but excedences above
HAT would have been few during the Tidal Epoch, because tidal
predictions (including the definition of HAT) were based on observed
tides during that period.

## Resampling

Create a function to allow us to use `map()` or `lapply()` to assemble
results.

The function `res_auto()` is not fully encapsulated, since it relies on
the existence of a dataframe called “epoch”, with data columns with
specific names.

``` r
res_auto <- function(slr, samp) {
  res <- numeric(samp)
  for (iter in seq(1, samp))
    res[[iter]] <- resample_once(epoch, Prediction, deviation, theDate, slr)
  return(res)
}
  
```

The following takes a couple of minutes to run. It simulates 1000 sets
of tide levels for each of four SLR scenarios.

``` r
set.seed(12345)
samp = 1000
resamp <- lapply((0:3 * 0.3048), function(x) res_auto(x, samp))
```

``` r
names(resamp) = c('No SLR', 'One Foot SLR', 'Two Foot SLR', 'Three Foot SLR')
resamp <- do.call(bind_cols, resamp)
```

``` r
resamp <- resamp %>% 
  summarize(across(everything(), mean, na.rm = TRUE)) %>%
  rowwise() %>%
  mutate(`One Foot`   = round(`One Foot SLR`/`No SLR`,2),
         `Two Foot`   = round(`Two Foot SLR`/`No SLR`,2),
         `Three Foot` = round(`Three Foot SLR`/`No SLR`,2))
resamp
#> # A tibble: 1 x 7
#> # Rowwise: 
#>   `No SLR` `One Foot SLR` `Two Foot SLR` `Three Foot SLR` `One Foot` `Two Foot`
#>      <dbl>          <dbl>          <dbl>            <dbl>      <dbl>      <dbl>
#> 1     6.04           61.9           191.             336.       10.2       31.7
#> # ... with 1 more variable: Three Foot <dbl>
```

``` r
v <- resamp$`One Foot` 
```

We can conclude that under this model, one foot of SLR will increase the
frequency of flooding by something on the order of a factor of 10.2. The
analysis is conservative, to the extent that it is based on sea level
observations during the tidal epoch, which ended twenty years ago. The
ratio of increased flooding, however, is likely to be similar.

## ARIMA

We use the same approach here. Again, `sim_auto()` is not fully
encapsulated.

``` r
sim_auto <- function(slr, samp) {
  res <- numeric(samp)
  for (iter in seq(1, samp))
    res[[iter]] <- sim_once(epoch, Prediction, theDate, slr)
  return(res)
}
  
```

The following takes several minutes to run.

``` r
set.seed(12345)
samp = 1000
simulates <- lapply((0:3 * 0.3048), function(x) sim_auto(x, samp))
```

``` r
names(simulates) = c('No SLR', 'One Foot SLR', 'Two Foot SLR', 'Three Foot SLR')
simulates <- do.call(bind_cols, simulates)
```

``` r
simulates_sum <- simulates %>%
  summarize(across(everything(), mean, na.rm = TRUE))%>%
  rowwise() %>%
  mutate(`One Foot`   = round(`One Foot SLR`/`No SLR`,2),
         `Two Foot`   = round(`Two Foot SLR`/`No SLR`,2),
         `Three Foot` = round(`Three Foot SLR`/`No SLR`,2))
simulates_sum
#> # A tibble: 1 x 7
#> # Rowwise: 
#>   `No SLR` `One Foot SLR` `Two Foot SLR` `Three Foot SLR` `One Foot` `Two Foot`
#>      <dbl>          <dbl>          <dbl>            <dbl>      <dbl>      <dbl>
#> 1     4.31           52.2           172.             319.       12.1       39.8
#> # ... with 1 more variable: Three Foot <dbl>

v <- simulates_sum$`One Foot`
```

This analysis suggests a one foot sea level rise would increase flooding
by a factor of about 12.1.

## Make a Nice Table

``` r
t <- rbind(slr_annual, resamp, simulates_sum)
row.names(t) <- c('Add SLR to tidal Epoch', 'Resampling Model', 'ARIMA Model')
#> Warning: Setting row names on a tibble is deprecated.
knitr::kable(t)
```

|                        |   No SLR | One Foot SLR | Two Foot SLR | Three Foot SLR | One Foot | Two Foot | Three Foot |
|:-----------------------|---------:|-------------:|-------------:|---------------:|---------:|---------:|-----------:|
| Add SLR to tidal Epoch | 2.105187 |     45.84045 |     161.2047 |       311.4098 |    21.77 |    76.58 |     147.92 |
| Resampling Model       | 6.038519 |     61.86961 |     191.3714 |       335.5146 |    10.25 |    31.69 |      55.56 |
| ARIMA Model            | 4.314739 |     52.18170 |     171.7737 |       318.9858 |    12.09 |    39.81 |      73.93 |

# Updated To Present Conditions

Those simulations reflect conditions that held during the Tidal Epoch,
from 1983 through 2001. Today’s sea levels are slightly higher. How
would taking that into account affect predictions of the frequency of
flooding and the relative increase in frequency of flooding we expect
with one foot of *additional* SLR?

Actual tidal levels today are several centimeters higher than during the
tidal epoch. With average SLR on the order of 2 mm per year, observed
elevations today should be on the order of
$2.0 \\frac{\\text{mm}}{\\text{yr}} \\times 19\\text{ yrs} = 38 \\text{ mm} \\approx 1.5 \\text{ in}$.

How much will our results differ if we start from a base elevation a few
inches higher? We can compare frequency of flooding with an extra 4 cm
of base elevation versus flooding an additional 4cm plus one foot of
SLR.

## Updated Direct Computation

Following the MGS method, but focused on the same period as our
simulations for comparison purposes. Note these results are VERY
different from results for the last ten years, and in fact make the
increase in flooding frequency look much more severe.

Note we are working in meters here.

``` r
HAT <- 3.640
slr_annual_2 <- epoch %>%
  mutate(slr_1 = .038 + MLLW + 1 * 0.3048,
         slr_2 = .038 + MLLW + 2 * 0.3048,
         slr_3 = .038 + MLLW + 3 * 0.3048,
         exceeds_0 = MLLW   > HAT, 
         exceeds_0.1 = 0.038 + MLLW  > HAT, 
         exceeds_1 = slr_1   > HAT,
         exceeds_2 = slr_2   > HAT,
         exceeds_3 = slr_3   > HAT) %>%
  group_by(theDate) %>%
  summarize(exceeded_0 = any(exceeds_0, na.rm = TRUE),
            exceeded_0.1 = any(exceeds_0.1, na.rm = TRUE),
            exceeded_1 = any(exceeds_1, na.rm = TRUE),
            exceeded_2 = any(exceeds_2, na.rm = TRUE),
            exceeded_3 = any(exceeds_3, na.rm = TRUE),
            .groups = 'drop') %>%
  
  summarize(days = sum(! is.na(exceeded_0)),
            `Tidal Epoch` = 365.25 * sum(exceeded_0)/days,
            `No SLR` = 365.25 * sum(exceeded_0.1)/days,
            `One Foot SLR` = 365.25 * sum(exceeded_1)/days,
            `Two Foot SLR` = 365.25 * sum(exceeded_2)/days,
            `Three Foot SLR` = 365.25 * sum(exceeded_3)/days,
            .groups = 'drop') %>%
  select(-days)
slr_annual_2
#> # A tibble: 1 x 5
#>   `Tidal Epoch` `No SLR` `One Foot SLR` `Two Foot SLR` `Three Foot SLR`
#>           <dbl>    <dbl>          <dbl>          <dbl>            <dbl>
#> 1          2.11     3.89           55.8           180.             324.
```

## Updated Simulation

``` r
levels <- 0.038 * c(0,1,1,1,1) + (c(0,0,1,2,3) * 0.3048)
simulates_2 <- lapply(levels, 
                    function(x) sim_auto(x, samp))
```

``` r
names(simulates_2) = c('Tidal Epoch', 'No SLR', 
                       'One Foot SLR', 'Two Foot SLR', 
                       'Three Foot SLR')
simulates_2 <- do.call(bind_cols, simulates_2)
```

``` r
simulation <- simulates_2 %>%
  summarize(across(everything(), c(mean, sd), na.rm = TRUE), .groups = 'drop')
simulation
#> # A tibble: 1 x 10
#>   `Tidal Epoch_1` `Tidal Epoch_2` `No SLR_1` `No SLR_2` `One Foot SLR_1`
#>             <dbl>           <dbl>      <dbl>      <dbl>            <dbl>
#> 1            4.34           0.487       6.75      0.569             62.6
#> # ... with 5 more variables: One Foot SLR_2 <dbl>, Two Foot SLR_1 <dbl>,
#> #   Two Foot SLR_2 <dbl>, Three Foot SLR_1 <dbl>, Three Foot SLR_2 <dbl>
```
