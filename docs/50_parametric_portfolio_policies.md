# Parametric Portfolio Policies



In this section, we introduce different portfolio performance measures to evaluate and compare different allocation strategies. For this purpose, we introduce a direct (and probably simplest) way to estimate optimal portfolio weights when stock characteristics are related to the stock’s expected return, variance, and covariance with other stocks: We parametrize weights as a function of the characteristics such that we maximize expected utility. The approach is feasible for large portfolio dimensions (such as the entire CRSP universe) and has been proposed by [@Brandt2009] in their influential paper *Parametric Portfolio Policies: Exploiting Characteristics in the Cross Section of Equity Returns*. 

## Data preparation

To get started, we load the monthly CRSP file which forms our investment universe. 

```r
library(tidyverse)
library(lubridate)
library(RSQLite)
```


```r
# Load data from database
tidy_finance <- dbConnect(SQLite(), "data/tidy_finance.sqlite", extended_types = TRUE)

crsp_monthly <- tbl(tidy_finance, "crsp_monthly") %>%
  collect()
```

For the purpose of performance evaluation, we need monthly market returns in order to be able to compute CAPM alphas which we retrieve from the Fama French 3 factor model data. 

```r
factors_ff_monthly <- tbl(tidy_finance, "factors_ff_monthly") %>%
  collect()
```

Next we create some characteristics that have been proposed in the literature to have an effect on the expected returns or expected variances (or even higher moments) of the return distribution: We record for each firm the lagged one-year return *(mom)* defined as the compounded return between months $t − 13$ and $t − 2$ and the firm's market equity *(me)*, defined as the log of the price per share times the number of shares outstanding


```r
crsp_monthly_lags <- crsp_monthly %>% 
  transmute(permno, 
            month_12 = month %m+% months(12), 
            month_1 = month %m+% months(1),
            altprc = abs(altprc))

crsp_monthly <- crsp_monthly %>% 
  inner_join(crsp_monthly_lags %>% select(-month_1), 
             by = c("permno", "month" = "month_12"), suffix = c("", "_12")) %>% 
  inner_join(crsp_monthly_lags %>% select(-month_12), 
             by = c("permno", "month" = "month_1"), suffix = c("", "_1"))

# Create characteristics for portfolio tilts
crsp_monthly <- crsp_monthly %>%
  group_by(permno) %>%
  mutate(
    momentum_lag = altprc_1 / altprc_12, # Gross returns TODO for SV: Do we need slider?
    size_lag = log(mktcap_lag)
  ) %>%
  drop_na(contains("lag"))
```

## Parametric Portfolio Policies
The basic idea of parametric portfolio weights is easy to explain: Suppose that at each date $t$ we have $N_t$ stocks in the investment universe, where each stock $i$ has return of $r_{i, t+1}$ and is associated with a vector of firm characteristics $x_{i, t}$ such as time-series momentum or the market capitalization. The investors problem is to choose portfolio weights $w_{i,t}$ to maximize the expected utility of the portfolio return:
$$\begin{align}
\max_{w} E_t\left(u(r_{p, t+1})\right) = E_t\left[u\left(\sum\limits_{i=1}^{N_t}w_{i,t}r_{i,t+1}\right)\right]
\end{align}$$
where $u(\cdot)$ denotes the utility function.
Where do the stock characteristics show up? We parametrize the optimal portfolio weights as a function of the stock characteristic $x_{i,t}$ with the following linear specification for the portfolio weights:
$$w_{i,t} = \bar{w}_{i,t} + \frac{1}{N_t}\theta'\hat{x}_{i,t}$$ where $\bar{w}_{i,t}$ is the weight of a benchmark portfolio (we use the value-weighted or naive portfolio in the application below), $\theta$ is a vector of coefficients which we are going to estimate and $\hat{x}_{i,t}$ are the characteristics of stick $i$, cross-sectionally standardized to have zero mean and unit standard deviation. Think of the portfolio strategy as a form of active portfolio management relative to a performance benchmark: Deviations from the benchmark portfolio are derived from the individual stock characteristics. Note that by construction the weights sum up to one as $\sum_{i=1}^{N_t}\hat{x}_{i,t} = 0$ due to the standardization. Note also that the coefficients are constant across assets and through time. The implicit assumption is that the characteristics fully capture all aspects of the joint distribution of returns that are relevant for forming optimal portfolios.       

We first implement the cross-sectional standardization for the entire CRSP universe. We also keep track of (lagged) relative market capitalization which will represent the value-weighted benchmark portfolio.

```r
crsp_monthly <- crsp_monthly %>%
  group_by(month) %>%
  mutate(
    n = n(), # number of traded assets N_t (benchmark for naive portfolio)
    relative_mktcap = mktcap_lag / sum(mktcap_lag), # Value weighting benchmark
    across(contains("lag"), ~ (. - mean(.)) / sd(.)), # standardization. Note: Code handles every column with "lag" as a characteristic)
  ) %>%
  ungroup() %>%
  select(-mktcap_lag, -altprc)
```

## Compute portfolio weights
Next we can move to optimal choices of $\theta$. We rewrite the optimization problem together with the weight parametrisation and can then estimate $\theta$ to maximize the objective function based on our sample 
$$\begin{align}
E_t\left(u(r_{p, t+1})\right) = \frac{1}{T}\sum\limits_{t=0}^{T-1}u\left(\sum\limits_{i=1}^{N_t}\left(\bar{w}_{i,t} + \frac{1}{N_t}\theta'\hat{x}_{i,t}\right)r_{i,t+1}\right).
\end{align}$$
The allocation strategy is simple because the number of parameters to estimate is small. Instead of a tedious specification of the $N_t$ dimensional vector of expected returns and the $N_t(N_t+1)/2$ free elements of the variance covariance, all we need to focus on in our application is the vector $\theta$ which contains only 2 elements in our application - the relative deviation from the benchmark due to *size* and due to *past returns*. 

To get a feeling on the performance of such an allocation strategy we start with an arbitrary vector $\theta$ - the next step is then to choose $\theta$ in an optimal fashion to maximize the objective function. 


```r
# Automatic detection of parameters and initialization of parameter vector theta
number_of_param <- sum(grepl("lag", crsp_monthly %>% colnames()))
theta <- rep(1.5, number_of_param) # We start with some arbitrary initial values for theta
names(theta) <- colnames(crsp_monthly)[grepl("lag", crsp_monthly %>% colnames())]
```

The following function computes the portfolio weights $\bar{w}_{i,t} + \frac{1}{N_t}\theta'\hat{x}_{i,t}$ according to our parametrization for a given value of $\theta$. Everything happens within a single pipeline, thus here goes a quick walk-through:

We first compute `characteristic_tilt`, the tilting values $\frac{1}{N_t}\theta'\hat{x}_{i, t}$ which resemble the deviation from the benchmark portfolio. Next, we compute the benchmark portfolio `weight_benchmark` which in principle can be any reasonable set of portfolio weights. In our case we choose either the value- or equal-weighted allocation. 
`weight_tilt` completes the picture and contains the final portfolio weights `weight_tilt = weight_benchmark + characteristic_tilt` which deviate from the benchmark portfolio depending on the stock characteristics.

The final few lines go a bit further and implement a simple version of a no-short sale constraint. While it is generally not straightforward to ensure portfolio weight constraints via the parametrization, here we simply normalize the portfolio weights such that they are enforced to be positive. Finally, we make sure that the normalized weights sum up to one again. We do so by  
$$w_{i,t}^+ = \frac{\max(0, w_{i,t})}{\sum\limits_{j=1}^{N_t}\max(0, w_{i,t})}.$$
Here we go to optimal portfolio weights in 20 lines. 

```r
compute_portfolio_weights <- function(theta,
                                      data,
                                      value_weighting = TRUE,
                                      allow_short_selling = TRUE) {
  data %>%
    group_by(month) %>%
    bind_cols(
      characteristic_tilt = data %>% # Computes theta'x / N_t
        transmute(across(contains("lag"), ~ . / n)) %>%
        as.matrix() %*% theta %>% as.numeric()
    ) %>%
    mutate(
      weight_benchmark = case_when( # Definition of the benchmark weight
        value_weighting == TRUE ~ relative_mktcap,
        value_weighting == FALSE ~ 1 / n
      ),
      weight_tilt = weight_benchmark + characteristic_tilt, # Parametric Portfolio Weights
      weight_tilt = case_when( # Short-sell constraint
        allow_short_selling == TRUE ~ weight_tilt,
        allow_short_selling == FALSE ~ pmax(0, weight_tilt)
      ),
      weight_tilt = weight_tilt / sum(weight_tilt) # Weights sum up to 1
    ) %>%
    ungroup()
}
```

Done! Next step is to compute the portfolio weights for a given vector $\theta$ at your convenience. In the example below we use the value weighted portfolio as a benchmark and allow negative portfolio weights.

```r
weights_crsp <- compute_portfolio_weights(theta,
  crsp_monthly,
  value_weighting = TRUE,
  allow_short_selling = TRUE
)
```

## Portfolio perfomance

Are these weights optimal in any way? Most likely not as we chose $\theta$ arbitrarily. In order to evaluate the performance of an allocation strategy, one can think of many different approaches. In their original paper, [@Brandt2009] focus on a simple evaluation of the hypothetical utility of an agent equipped with a power utility function $u_\gamma(r) = \frac{(1 + r)^\gamma}{1-\gamma}$, where $\gamma$ is the risk aversion factor. 

```r
power_utility <- function(r, gamma = 5) { 
  (1 + r)^(1 - gamma) / (1 - gamma)
}
```

No doubt, there are many more ways to evaluate a portfolio. The function below provides a summary of all kinds of interesting measures which all can be considered relevant. 
Do we need all these evaluation measures? It depends: The original paper only cares about
expected utility in order to choose $\theta$. But if you want to choose optimal values that achieve the highest performance while putting some constraints on your portfolio weights it is helpful to have everything in one function.


```r
evaluate_portfolio <- function(weights_crsp,
                               full_evaluation = TRUE) {

  evaluation <- weights_crsp %>%
    group_by(month) %>%
    # Compute monthly portfolio returns
    summarise(
      return_tilt = weighted.mean(ret_excess, weight_tilt),
      return_benchmark = weighted.mean(ret_excess, weight_benchmark)
    ) %>%
    pivot_longer(-month, values_to = "portfolio_return", names_to = "model") %>%
    group_by(model) %>%
    left_join(factors_ff_monthly, by = "month") %>% # FF data to compute alpha
    summarise(tibble(
      "Expected utility" = mean(power_utility(portfolio_return)),
      "Average return" = 100 * mean(12 * portfolio_return),
      "SD return" = 100 * sqrt(12) * sd(portfolio_return),
      "Sharpe ratio" = mean(portfolio_return) / sd(portfolio_return),
      "CAPM alpha" = coefficients(lm(portfolio_return ~ mkt_excess))[1],
      "Market beta" = coefficients(lm(portfolio_return ~ mkt_excess))[2]
    )) %>%
    mutate(model = gsub("return_", "", model)) %>%
    pivot_longer(-model, names_to = "measure") %>%
    pivot_wider(names_from = model, values_from = value)

  if (full_evaluation) { # additional values based on the portfolio weights
    weight_evaluation <- weights_crsp %>%
      select(month, contains("weight")) %>%
      pivot_longer(-month, values_to = "weight", names_to = "model") %>%
      group_by(model, month) %>%
      transmute(tibble(
        "Absolute weight" = abs(weight),
        "Max. weight" = max(weight),
        "Min. weight" = min(weight),
        "Avg. sum of negative weights" = -sum(weight[weight < 0]),
        "Avg. fraction of negative weights" = sum(weight < 0) / n()
      )) %>%
      group_by(model) %>%
      summarise(across(-month, ~ 100 * mean(.))) %>%
      mutate(model = gsub("weight_", "", model)) %>%
      pivot_longer(-model, names_to = "measure") %>%
      pivot_wider(names_from = model, values_from = value)
    evaluation <- bind_rows(evaluation, weight_evaluation)
  }
  return(evaluation)
}
```

Let's take a look at the different portfolio strategies and evaluation measures.

```r
evaluate_portfolio(weights_crsp) %>% 
  kableExtra::kable(digits = 3)
```

<table>
 <thead>
  <tr>
   <th style="text-align:left;"> measure </th>
   <th style="text-align:right;"> benchmark </th>
   <th style="text-align:right;"> tilt </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Expected utility </td>
   <td style="text-align:right;"> -0.249 </td>
   <td style="text-align:right;"> -0.261 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Average return </td>
   <td style="text-align:right;"> 6.858 </td>
   <td style="text-align:right;"> -0.051 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> SD return </td>
   <td style="text-align:right;"> 15.348 </td>
   <td style="text-align:right;"> 20.237 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Sharpe ratio </td>
   <td style="text-align:right;"> 0.129 </td>
   <td style="text-align:right;"> -0.001 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> CAPM alpha </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> -0.005 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Market beta </td>
   <td style="text-align:right;"> 0.992 </td>
   <td style="text-align:right;"> 0.904 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Absolute weight </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 0.061 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Max. weight </td>
   <td style="text-align:right;"> 3.516 </td>
   <td style="text-align:right;"> 3.647 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Min. weight </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> -0.139 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Avg. sum of negative weights </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 72.910 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Avg. fraction of negative weights </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 49.366 </td>
  </tr>
</tbody>
</table>

The value weighted portfolio delivers an annualized return of above 6 percent and clearly outperforms the tilted portfolio, irrespective of whether we evaluate expected utility, the Sharpe ratio or the CAPM alpha. We can conclude the the market beta is close to one for both strategies (naturally almost identically 1 for the value-weighted benchmark portfolio). When it comes to the distribution of the portfolio weights, we see that the benchmark portfolio weight takes less extreme positions (lower average absolute weights and lower maximum weight). By definition, the value-weighted benchmark does not take any negative positions, while the tilted portfolio also takes short positions. 

## Optimal parameter choice
Next we move to a choice of $\theta$ that actually aims to to improve some (or all) of the performance measures. We first define a helper function `compute_objective_function` which is then passed to R's optimization schemes. 

```r
compute_objective_function <- function(theta,
                                       data,
                                       objective_measure = "Expected utility",
                                       value_weighting,
                                       allow_short_selling) {
  processed_data <- compute_portfolio_weights(
    theta,
    data,
    value_weighting,
    allow_short_selling
  )

  objective_function <- evaluate_portfolio(processed_data, full_evaluation = FALSE) %>%
    filter(measure == objective_measure) %>%
    pull(tilt)

  return(-objective_function)
}
```
You may wonder why we return the negative value of the objective function. This is simply due to the common convention for optimization procedures to search minima as a default. By minimizing the negative value of the objective function we will get the maximum value as a result.
Optimization in R in its most basic form is done by `optim`. As main inputs, the function requires an initial "guess" of the parameters, and the function to minimize. We are fully equipped to compute the optimal values of $\hat\theta$ which maximize hypothetical expected utility of the investor. 


```r
optimal_theta <- optim(
  par = theta, # Initial vector of thetas (can be any value)
  compute_objective_function,
  objective_measure = "Expected utility",
  data = crsp_monthly,
  value_weighting = TRUE,
  allow_short_selling = TRUE
)

optimal_theta$par # Optimal values
```

```
## momentum_lag     size_lag 
##    0.5889182   -2.0715581
```
The chosen values of $\theta$ are easy to interpret on an intuitive basis: Expected utility increases by tilting weights from the value weighted portfolio towards smaller stocks (negative coefficient for size) and towards past winners (positive value for momentum). 

## More model specifications
A final open question is then: How does the portfolio perform for different model specifications? For that purpose we compute the performance of a number of different modelling choices based on the entire CRSP sample. The few lines below perform all the heavy lifting.


```r
full_model_grid <- expand_grid(
  value_weighting = c(TRUE, FALSE),
  allow_short_selling = c(TRUE, FALSE),
  data = list(crsp_monthly)
) %>%
  mutate(optimal_theta = pmap(
    .l = list(
      data,
      value_weighting,
      allow_short_selling
    ),
    .f = ~ optim(
      par = theta,
      compute_objective_function,
      data = ..1,
      objective_measure = "Expected utility",
      value_weighting = ..2,
      allow_short_selling = ..3)$par
  ))
```

Finally, we can compare the results. The table below shows summary statistics for all possible combinations: Equal- or Value-weighted benchmark portfolio, with or without short-selling constraints, tilted towards maximizing expected utility. 

```r
performance_table <- full_model_grid %>%
  mutate(
    processed_data = pmap(
      .l = list(optimal_theta, data, value_weighting, allow_short_selling),
      .f = ~ compute_portfolio_weights(..1, ..2, ..3, ..4)
    ),
    portfolio_evaluation = map(processed_data, evaluate_portfolio, full_evaluation = TRUE)
  ) %>%
  select(value_weighting, allow_short_selling, portfolio_evaluation) %>%
  unnest(portfolio_evaluation)

performance_table %>%
  rename(
    " " = benchmark,
    Optimal = tilt
  ) %>%
  mutate(
    value_weighting = case_when(
      value_weighting == TRUE ~ "VW",
      value_weighting == FALSE ~ "EW"
    ),
    allow_short_selling = case_when(
      allow_short_selling == TRUE ~ "",
      allow_short_selling == FALSE ~ "(no s.)"
    )
  ) %>%
  pivot_wider(
    names_from = value_weighting:allow_short_selling,
    values_from = " ":Optimal,
    names_glue = "{value_weighting} {allow_short_selling} {.value} "
  ) %>%
  select(measure, `EW    `, `VW    `, sort(contains("Optimal"))) %>%
  kableExtra::kable(digits = 3)
```

<table>
 <thead>
  <tr>
   <th style="text-align:left;"> measure </th>
   <th style="text-align:right;"> EW     </th>
   <th style="text-align:right;"> VW     </th>
   <th style="text-align:right;"> VW  Optimal  </th>
   <th style="text-align:right;"> VW (no s.) Optimal  </th>
   <th style="text-align:right;"> EW  Optimal  </th>
   <th style="text-align:right;"> EW (no s.) Optimal  </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Expected utility </td>
   <td style="text-align:right;"> -0.250 </td>
   <td style="text-align:right;"> -0.249 </td>
   <td style="text-align:right;"> -0.246 </td>
   <td style="text-align:right;"> -0.247 </td>
   <td style="text-align:right;"> -0.250 </td>
   <td style="text-align:right;"> -0.250 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Average return </td>
   <td style="text-align:right;"> 10.456 </td>
   <td style="text-align:right;"> 6.858 </td>
   <td style="text-align:right;"> 14.697 </td>
   <td style="text-align:right;"> 13.413 </td>
   <td style="text-align:right;"> 13.014 </td>
   <td style="text-align:right;"> 8.052 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> SD return </td>
   <td style="text-align:right;"> 20.333 </td>
   <td style="text-align:right;"> 15.348 </td>
   <td style="text-align:right;"> 20.342 </td>
   <td style="text-align:right;"> 19.584 </td>
   <td style="text-align:right;"> 22.402 </td>
   <td style="text-align:right;"> 17.096 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Sharpe ratio </td>
   <td style="text-align:right;"> 0.148 </td>
   <td style="text-align:right;"> 0.129 </td>
   <td style="text-align:right;"> 0.209 </td>
   <td style="text-align:right;"> 0.198 </td>
   <td style="text-align:right;"> 0.168 </td>
   <td style="text-align:right;"> 0.136 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> CAPM alpha </td>
   <td style="text-align:right;"> 0.002 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.007 </td>
   <td style="text-align:right;"> 0.005 </td>
   <td style="text-align:right;"> 0.005 </td>
   <td style="text-align:right;"> 0.001 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Market beta </td>
   <td style="text-align:right;"> 1.135 </td>
   <td style="text-align:right;"> 0.992 </td>
   <td style="text-align:right;"> 0.991 </td>
   <td style="text-align:right;"> 1.045 </td>
   <td style="text-align:right;"> 1.122 </td>
   <td style="text-align:right;"> 1.077 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Absolute weight </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 0.040 </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 0.027 </td>
   <td style="text-align:right;"> 0.025 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Max. weight </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 3.516 </td>
   <td style="text-align:right;"> 3.331 </td>
   <td style="text-align:right;"> 2.643 </td>
   <td style="text-align:right;"> 0.367 </td>
   <td style="text-align:right;"> 0.297 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Min. weight </td>
   <td style="text-align:right;"> 0.025 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> -0.046 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> -0.042 </td>
   <td style="text-align:right;"> 0.000 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Avg. sum of negative weights </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 31.098 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 4.040 </td>
   <td style="text-align:right;"> 0.000 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Avg. fraction of negative weights </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 39.368 </td>
   <td style="text-align:right;"> 0.000 </td>
   <td style="text-align:right;"> 10.409 </td>
   <td style="text-align:right;"> 0.000 </td>
  </tr>
</tbody>
</table>

The results indicate first that the average annualized Sharpe ratio of the equal weighted portfolio exceeded the Sharpe ratio of the value weighted benchmark portfolio. Nevertheless, starting with the value weighted portfolio as a benchmark and tilting optimally with respect to momentum and small stocks yields the highest Sharpe ratio across all specifications. Imposing no short-sale constraints does not improve the performance of the portfolios in our application.

## Exercises

1. How do the estimated parameters $\hat\theta$ and the portfolio performance change if your objective is to maximize the Sharpe ratio instead of hypothetical expected utility?
1. The code above is very flexible in the sense that you can easily add new firm characteristics. Construct a new characteristic and evaluate the corresponding coefficient $\hat\theta_i$. 
1. Tweak the function `optimal_theta`such that you can impose additional performance constraints in order to determine $\hat\theta$ which maximizes expected utility under the constraint that the market beta is below 1.
1. Does the portfolio performance resemble a realistic out-of-sample backtesting procedure? Verify the robustness of the results by first estimating $\hat\theta$ based on *past data* only and then use more recent periods to evaluate the actual portfolio performance. 
1. By formulating the portfolio problem as a statistical estimation problem, you can easily obtain standard errors for the coefficients of the weight function. [@Brandt2009] provide the relevant derivations in their paper in Equation (10). Implement a small function that computes standard errors for $\hat\theta$.