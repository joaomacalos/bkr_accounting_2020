Context plots
================
João Pedro S. Macalós
2/10/2020

The context plots requires two series: the BRL/USD exchange rate and the
annual growth rate of the Brazilian GDP. The first is downloaded from
the IMF database and the second from the World Bank database:

Load `tidyverse` and
    `lubridate`:

``` r
library(tidyverse)
```

    ## ── Attaching packages ────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse 1.3.0 ──

    ## ✓ ggplot2 3.2.1     ✓ purrr   0.3.3
    ## ✓ tibble  2.1.3     ✓ dplyr   0.8.3
    ## ✓ tidyr   1.0.0     ✓ stringr 1.4.0
    ## ✓ readr   1.3.1     ✓ forcats 0.4.0

    ## ── Conflicts ───────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(lubridate)
```

    ## 
    ## Attaching package: 'lubridate'

    ## The following object is masked from 'package:base':
    ## 
    ##     date

The following functions are wrappers that download and clean the the IMF
and WB series:

``` r
ifs_getA <- function(id, country = list_em2, start = 2000) {
  imfr::imf_data(database_id = "IFS", indicator = id, freq = "A", country = country, start = start) %>%
    as_tibble() %>%
    mutate(date = ymd(year, truncated = 2)) %>%
    select(iso2c, date, id) %>%
    gather(key, value, -date, -iso2c)
}

wb_getA <- function(id, country = list_em2, start = 2000, end = 2018) {
  wbstats::wb(country = country, indicator = id, freq = 'Y', startdate = start, enddate = end) %>% 
    as_tibble %>%
    select(iso2c, date, value) %>%
    mutate(date = ymd(as.numeric(date), truncated = 2)) %>%
    arrange(date) %>% arrange(iso2c)
}
```

``` r
exr_br <- ifs_getA('ENDA_XDC_USD_RATE', 'BR', 2004) %>% select(date, exr = value)
gdp_br <- wb_getA('NY.GDP.MKTP.KD.ZG', 'BR', start = 2004) %>% select(date, gdp = value)
```

Join the variables:

``` r
context_vars <- inner_join(gdp_br, exr_br) %>%
  gather(key, value, -date)

#write_tsv(context_vars, 'context_vars_raw.tsv')
```

``` r
context_vars %>%
  mutate(key = if_else(key == 'exr', 'BRL/USD Exchange rate', 'Gross domestic product (% change)')) %>%
  ggplot(aes(x=date, y=value, color = key)) +
  geom_line(size = 2) +
  scale_color_brewer(palette = 'Dark2') +
  facet_wrap(~key, nrow =  2, scales = 'free') +
  ggthemes::theme_economist() +
  theme(legend.position = 'none') +
  labs(x = '', y = '', caption = 'Sources: IMF-IFS and WB') +
  scale_x_date(breaks = '2 years', date_labels = '%Y')
```

![](context_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->
