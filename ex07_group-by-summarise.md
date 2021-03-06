Work on groups of rows via dplyr::group\_by() + summarise()
================
Jenny Bryan
2018-04-11

What if you need to work on groups of rows? Such as the groups induced
by the levels of a factor.

You do not need to … split the data frame into mini-data-frames, loop
over them, and glue it all back together.

Instead, use `dplyr::group_by()`, followed by `dplyr::summarize()`, to
compute group-wise summaries.

``` r
library(tidyverse)

iris %>%
  group_by(Species) %>%
  summarise(pl_avg = mean(Petal.Length), pw_avg = mean(Petal.Width))
#> # A tibble: 3 x 3
#>   Species    pl_avg pw_avg
#>   <fct>       <dbl>  <dbl>
#> 1 setosa       1.46  0.246
#> 2 versicolor   4.26  1.33 
#> 3 virginica    5.55  2.03
```

What if you want to return summaries that are not just a single number?

This does not “just work”.

``` r
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = quantile(Petal.Length, c(0.25, 0.5, 0.75)))
#> Error in summarise_impl(.data, dots): Column `pl_qtile` must be length 1 (a summary value), not 3
```

Solution: package as a length-1 list that contains 3 values, creating a
list-column.

``` r
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = list(quantile(Petal.Length, c(0.25, 0.5, 0.75))))
#> # A tibble: 3 x 2
#>   Species    pl_qtile 
#>   <fct>      <list>   
#> 1 setosa     <dbl [3]>
#> 2 versicolor <dbl [3]>
#> 3 virginica  <dbl [3]>
```

Q from
[@jcpsantiago](https://twitter.com/jcpsantiago/status/983997363298717696)
via Twitter: How would you unnest so the final output is a data frame
with a factor column `quantile` with levels “25%”, “50%”, and “75%”?

A: I would `map()` `tibble::enframe()` on the new list column, to
convert each entry from named list to a two-column data frame. Then use
`tidyr::unnest()` to get rid of the list column and return to a simple
data frame and, if you like, convert `quantile` into a factor.

``` r
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = list(quantile(Petal.Length, c(0.25, 0.5, 0.75)))) %>%
  mutate(pl_qtile = map(pl_qtile, enframe, name = "quantile")) %>%
  unnest() %>%
  mutate(quantile = factor(quantile))
#> # A tibble: 9 x 3
#>   Species    quantile value
#>   <fct>      <fct>    <dbl>
#> 1 setosa     25%       1.40
#> 2 setosa     50%       1.50
#> 3 setosa     75%       1.58
#> 4 versicolor 25%       4.00
#> 5 versicolor 50%       4.35
#> 6 versicolor 75%       4.60
#> 7 virginica  25%       5.10
#> 8 virginica  50%       5.55
#> 9 virginica  75%       5.88
```

If something like this comes up a lot in an analysis, you could package
the key “moves” in a function, like so:

``` r
enquantile <- function(x, ...) {
  qtile <- enframe(quantile(x, ...), name = "quantile")
  qtile$quantile <- factor(qtile$quantile)
  list(qtile)
}
```

This makes repeated downstream usage more concise.

``` r
iris %>%
  group_by(Species) %>%
  summarise(pl_qtile = enquantile(Petal.Length, c(0.25, 0.5, 0.75))) %>%
  unnest()
#> # A tibble: 9 x 3
#>   Species    quantile value
#>   <fct>      <fct>    <dbl>
#> 1 setosa     25%       1.40
#> 2 setosa     50%       1.50
#> 3 setosa     75%       1.58
#> 4 versicolor 25%       4.00
#> 5 versicolor 50%       4.35
#> 6 versicolor 75%       4.60
#> 7 virginica  25%       5.10
#> 8 virginica  50%       5.55
#> 9 virginica  75%       5.88
```
