
<!-- README.md is generated from README.Rmd. Please edit that file -->

# jds-alignment-analysis

<!-- badges: start -->
<!-- badges: end -->

This repository contains data and code for the paper [Evaluating the
Alignment of a Data Analysis between Analyst and
Audience](https://arxiv.org/abs/2312.07616) by Lucy D’Agostino McGowan,
Roger D. Peng, Stephanie C. Hicks.

### Simulation for Section 5.1

``` r
library(tidyverse)
library(DirichletReg)

### Small amount of total resources to allocate ----
alpha_0 <- 1

set.seed(1)
# sample analyst i from field fi and get their deltas for principles
sample_deltas <- mvtnorm::rmvnorm(1, c(0, 0), matrix(c(.1, 0, 0, .1), ncol = 2))
data_analyst <- tibble(
  delta_p2 = sample_deltas[1],
  delta_p3 = sample_deltas[2],
  lambda_p2 = 0.3,
  lambda_p3 = 0.5,
  phi_2 = 0.2,
  phi_3 = 0.6,
  mu_2 = exp(lambda_p2 + delta_p2 + phi_2) / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  mu_3 = exp(lambda_p3 + delta_p3 + phi_3) / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  mu_1 = 1 / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  alpha_0 = alpha_0, # total resource being allocated, higher values mean less variability
  alpha_p2 = mu_2 * alpha_0,
  alpha_p1 = mu_1 * alpha_0,
  alpha_p3 = mu_3 * alpha_0
)

# N is the number of realizations of analysis A done by analyst i from field fi
n <- 100
w_analyst <- rdirichlet(n, c(data_analyst$alpha_p1, data_analyst$alpha_p2, data_analyst$alpha_p3))


w_analyst_df <- tibble(
  `Principle 1` = w_analyst[,1],
  `Principle 2` = w_analyst[,2],
  `Principle 3` = w_analyst[,3]) |>
  pivot_longer(`Principle 1`:`Principle 3`)

## Figure 2a
p_a <- ggplot(w_analyst_df, aes(x = name, y = value)) + 
  geom_boxplot() + 
  geom_jitter(alpha = 0.5)  + 
  theme_minimal() + 
  ylab("W") + 
  xlab("") + 
  ylim(c(0, 1))


### Larger amount of total resources to allocate ----
alpha_0 <- 100

set.seed(1)
# sample analyst i from field fi and get their deltas for principles
sample_deltas <- mvtnorm::rmvnorm(1, c(0, 0), matrix(c(.1, 0, 0, .1), ncol = 2))
data_analyst <- tibble(
  delta_p2 = sample_deltas[1],
  delta_p3 = sample_deltas[2],
  lambda_p2 = 0.3,
  lambda_p3 = 0.5,
  phi_2 = 0.2,
  phi_3 = 0.6,
  mu_2 = exp(lambda_p2 + delta_p2 + phi_2) / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  mu_3 = exp(lambda_p3 + delta_p3 + phi_3) / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  mu_1 = 1 / (1 + exp(lambda_p3 + delta_p3 + phi_3) + exp(lambda_p2 + delta_p2 + phi_2)),
  alpha_0 = alpha_0, # total resource being allocated, higher values mean less variability
  alpha_p2 = mu_2 * alpha_0,
  alpha_p1 = mu_1 * alpha_0,
  alpha_p3 = mu_3 * alpha_0
)

# N is the number of realizations of analysis A done by analyst i from field fi
n <- 100
w_analyst <- rdirichlet(n, c(data_analyst$alpha_p1, data_analyst$alpha_p2, data_analyst$alpha_p3))

w_analyst_df <- tibble(
  `Principle 1` = w_analyst[,1],
  `Principle 2` = w_analyst[,2],
  `Principle 3` = w_analyst[,3]) |>
  pivot_longer(`Principle 1`:`Principle 3`)

## Figure 2b
p_b <- ggplot(w_analyst_df, aes(x = name, y = value)) + 
  geom_boxplot() + 
  geom_jitter(alpha = 0.5)  + 
  theme_minimal() + 
  ylab("W") + 
  xlab("") + 
  ylim(c(0, 1))
```

### Case Study from Section 5.3

``` r
library(tidyverse)
dat <- read_csv("data.csv")
dat <- dat |>
  mutate(ind_total = rowSums(across(contains("individual"))),
         across(contains("individual"), ~.x / ind_total),
         grp_total = rowSums(across(contains("group"))),
         across(contains("group"), ~.x / grp_total),
         cons_total = rowSums(across(contains("consumer"))),
         across(contains("consumer"), ~.x / cons_total))


group_means <- dat |>
  mutate(grp = glue::glue("Group {grp}")) |>
  group_by(grp) |>
  summarise(across(matching_group:reproducibility_group,
                   ~ mean(.x, na.rm = TRUE))) |>
  pivot_longer(
    cols = matching_group:reproducibility_group,
    names_to = c(".value", "part"),
    names_pattern = "(.*)_(.*)"
  ) |>
  pivot_longer(
    cols = matching:reproducibility
  )
dat |>
  mutate(grp = glue::glue("Group {grp}")) |>
  pivot_longer(
    cols = matching_individual:reproducibility_consumer,
    names_to = c(".value", "part"),
    names_pattern = "(.*)_(.*)"
  ) |>
  pivot_longer(
    cols = matching:reproducibility
  ) |>
  filter(part != "group") |>
  mutate(part = case_when(part == "individual" ~ "baseline",
                          part == "consumer" ~ "resolution")) -> dat_long

## Figure 4 ----

dat_long |>
  ggplot(aes(x = part, y = value)) +
  geom_line(aes(group = ID)) +
  geom_point(alpha = 0.5) +
  geom_point(data = group_means, aes(x = "baseline", y = value), color = "cornflower blue", size = 2) +
  facet_grid(vars(grp), vars(name)) +
  xlab("") +
  ylab("W") + 
  theme_minimal()
```

![](README_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r

## Table 1 ----

dat |>
  mutate(grp = glue::glue("Group {grp}")) |>
  group_by(grp) |>
  summarise(across(matching_group:reproducibility_group,
                   ~ mean(.x, na.rm = TRUE)),
            across(matching_consumer:reproducibility_consumer,
                   ~ mean(.x, na.rm = TRUE)),
            diff_matching = matching_group - matching_consumer,
            diff_exhaustive = exhaustive_group - exhaustive_consumer,
            diff_skeptical = skeptical_group - skeptical_consumer,
            diff_secondorder = secondorder_group - secondorder_consumer,
            diff_clarity = clarity_group - clarity_consumer,
            diff_reproducibility = reproducibility_group - reproducibility_consumer
            ) |>
  select(grp, starts_with("diff")) |>
  knitr::kable(digits = 3)
```

| grp     | diff_matching | diff_exhaustive | diff_skeptical | diff_secondorder | diff_clarity | diff_reproducibility |
|:--------|--------------:|----------------:|---------------:|-----------------:|-------------:|---------------------:|
| Group 1 |        -0.036 |          -0.048 |          0.003 |            0.093 |       -0.009 |               -0.003 |
| Group 2 |         0.009 |           0.007 |         -0.017 |            0.006 |       -0.014 |                0.009 |
| Group 3 |         0.000 |           0.000 |          0.000 |            0.000 |        0.000 |                0.000 |
| Group 4 |         0.013 |          -0.031 |         -0.028 |           -0.027 |        0.013 |                0.059 |
| Group 5 |        -0.048 |          -0.011 |          0.038 |            0.019 |       -0.009 |                0.011 |
| Group 6 |         0.000 |           0.000 |          0.000 |            0.000 |        0.000 |                0.000 |
| Group 7 |        -0.002 |          -0.022 |         -0.002 |            0.019 |       -0.007 |                0.013 |
