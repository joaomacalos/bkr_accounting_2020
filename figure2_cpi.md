Consumer Price Indices
================
João Pedro S. Macalós
2/9/2020

The objective of this notebook is to show how the Figure 2 of the paper
“Does the accounting framework affect the operational capacity of the
central bank? Lessons from the Brazilian experience” was built, step by
step. The collection and cleaning steps are presented but not evaluated.
Static version of data is used at the end of the notebook for future
reproducibility of the paper.

This version of the data was downloaded in February 2020.

The first step is to load the `tidyverse` and `lubridate` packages that
are used extensively for cleaning the
    data:

``` r
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse 1.3.0 ──

    ## ✓ ggplot2 3.3.0           ✓ purrr   0.3.4      
    ## ✓ tibble  3.0.1           ✓ dplyr   0.8.99.9003
    ## ✓ tidyr   1.1.0           ✓ stringr 1.4.0      
    ## ✓ readr   1.3.1           ✓ forcats 0.5.0

    ## ── Conflicts ────────────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
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

The following functions are going to be useful:

``` r
# Not run
# Download and clean data from the IMF
ifs_getA <- function(id, country = list_em2) {
  imfr::imf_data(database_id = "IFS", indicator = id, freq = "A", country = country, start = 2000) %>%
    as_tibble() %>%
    mutate(date = ymd(year, truncated = 2)) %>%
    select(iso2c, date, id) %>%
    gather(key, value, -date, -iso2c)
}
```

``` r
# Not in function
'%!in%' <- function(x,y)!('%in%'(x,y))
```

## Downloading the data

Required steps:

1.  Selection of the newly integrated economies (NIEs).
2.  Download NIEs data.
3.  Download the data for Euro area countries plus Japan, United States
    and United Kingdom.

## NIEs

### List of NIEs

The list of countries selected for this study is based on the Wikipedia
entry of [Emerging
Markets](https://en.wikipedia.org/wiki/Emerging_market).

``` r
# Not run
url_emes <- xml2::read_html("https://en.wikipedia.org/wiki/Emerging_market")

list_emes_wiki <- url_emes %>% rvest::html_nodes(xpath = "/html/body/div[3]/div[3]/div[4]/div/table[1]") %>% rvest::html_table()

list_em_wiki <- list_emes_wiki[[1]]  %>%
  filter(Country != "Greece") %>%
  pull(Country)

list_em2 <- countrycode::countrycode(list_em_wiki, 'country.name', 'iso2c')
```

### Download

``` r
# Not run
cpi_nies <- ifs_getA('PCPI_IX', country = list_em2)
#write_tsv(cpi_nies, 'cpi_nies_raw.tsv')
```

Check the data:

``` r
# Not run
cpi_nies %>%
  filter(date >= '2005-01-01') %>%
  group_by(iso2c) %>%
  count() %>%
  arrange(n)
```

Argentina is missing completely and there is not enough data about
Taiwan and Venezuela.

``` r
# Not run
BIS_datasets <- BIS::get_datasets()
bis_cpi <- BIS::get_bis(BIS_datasets$url[BIS_datasets$name == "Consumer prices"])
```

Data from Argentina can be obtained in the BIS database:

``` r
# Not run
bis_cpi_missing <- bis_cpi %>%
  filter(freq == 'A',
         unit_of_measure == 'Index, 2010 = 100') %>%
  filter(ref_area %in% list_em2) %>%
  mutate(date = as.numeric(date)) %>%
  filter(date > 2004) %>%
  select(ref_area, date, obs_value) %>%
  set_names('iso2c', 'date', 'value') %>%
  filter(iso2c %in% c('AR', 'TW', 'VE', 'AE', 'IR')) %>%
  mutate(date = ymd(date, truncated = 2))

#write_tsv(bis_cpi_missing, 'cpi_nies_missing_raw.tsv')
```

Check the countries with the higher inflation rates in 2018:

``` r
# Not run
cpi_nies %>%
  bind_rows(bis_cpi_missing) %>%
  filter(iso2c %!in% c('VE', 'TW')) %>%
  select(-key) %>%
  filter(date == '2018-01-01') %>%
  arrange(desc(value)) %>% 
  slice(1:4)
```

To consolidate the data, Venezuela, Taiwan, and Iran are deleted due to
insufficient data (less than 14 observations). Furthermore, Argentina,
Egypt, Ukraine and Nigeria are deleted since they are the 4 highest
cases of consumer price indices in 2018.

``` r
# Not run
cpi_nies2 <-
  cpi_nies %>%
  filter(iso2c != 'AE') %>%
  bind_rows(bis_cpi_missing) %>%
  filter(iso2c %!in% c('VE', 'TW', 'AR', 'EG', 'UA', 'NG', 'IR')) %>%
  arrange(iso2c)

# AE is added back together with AR
```

``` r
# Not run
cpi_nies3 <- cpi_nies2 %>%
  select(-key) %>%
  spread(iso2c, value) %>%
  filter(date >= '2005-01-01', date < '2019-01-01') %>%
  mutate_at(vars(-date), list(~ . * 100 / (.[1])))
```

## Reserves’ currencies countries

The consumer price indices (CPI) of the NIEs are compared to the CPIs of
the countries that issue the currencies that denominate the majority of
the global international reserves, i.e., United States, Europe, Japan
and United Kingdom.

The CPI inside the Eurozone is calculated as the average CPI between the
countries inside the Eurozone in each year starting from 2005. To get
this series, the first step is to find a list of the countries inside
the Eurozone in each, a list that can be found in the Wikipedia entry
for the [Eurozone](https://en.wikipedia.org/wiki/Eurozone):

``` r
# Not run
url_eurozone <- xml2::read_html('https://en.wikipedia.org/wiki/Eurozone')
table_eurozone <- url_eurozone %>% 
  rvest::html_nodes(xpath = "/html/body/div[3]/div[3]/div[4]/div/table[3]") %>% 
  rvest::html_table()

list_eurozone <- 
  table_eurozone[[1]] %>%
  as_tibble() %>%
  mutate(Adopted = str_remove_all(Adopted, pattern = "\\[[0-9]{2}\\]$")) %>%
  select(State, Adopted) %>%
  mutate(date_adopted = ymd(Adopted)) %>%
  drop_na %>%
  mutate(iso2c = countrycode::countrycode(State, 'country.name', 'iso2c')) %>%
  mutate(inside = TRUE) %>%
  select(date_adopted, iso2c)
```

``` r
# Not run
list_eurozone
```

With a list that includes the date of adoption of the Euro it is
possible to proceed to the download of the data:

``` r
# Not run
cpi_euro <- ifs_getA('PCPI_IX', country = list_eurozone$iso2c)
#write_tsv(cpi_euro, 'cpi_euro_raw.tsv')
```

The next step is to summarize the data by year. In order to do that, we
filter out the countries prior to the adoption of the Euro:

``` r
# Not run
cpi_euro2 <-
  cpi_euro %>%
  full_join(list_eurozone, by='iso2c') %>%
  mutate(inside = if_else(date >= date_adopted, TRUE, FALSE)) %>%
  filter(inside == TRUE) %>%
  group_by(date) %>%
  summarize(eurozone = mean(value)) %>%
  gather(iso2c, value, -date)
```

This data must be joined to the data on the United States, Japan, and
the United Kingdom:

``` r
# Not run
cpi_usjpuk <- ifs_getA('PCPI_IX', country = c('US', 'JP', 'GB'))
#write_tsv(cpi_usjpuk, 'cpi_usjpuk_raw.tsv')
```

Consolidate the data for the reserve’ currencies countries:

``` r
# Not run
cpi_res <- cpi_usjpuk %>%
  select(-key) %>%
  bind_rows(cpi_euro2) %>%
  filter(date >= '2005-01-01')
```

``` r
# Not run
cpi_res2 <-
  cpi_res %>%
  spread(iso2c, value) %>%
  filter(date < '2019-01-01') %>%
  mutate_at(vars(-date), list(~ . * 100 / (.[1])))
```

#### Consolidate the data and save a backup file:

Here the data must be consolidated (already in the long format) and a
backup file is saved for future use:

``` r
# Not run
cpi_consolidated <- cpi_nies3 %>%
  left_join(cpi_res2) %>%
  gather(iso2c, value, -date)

#write_tsv(cpi_consolidated, 'cpi_consolidated.tsv')
```

## Figure:

Before building the Figure, a grouping variable `classification` is
added:

``` r
# Not run
cpi_consolidated <- cpi_consolidated %>%
  mutate(classification = if_else(iso2c %in% c(list_em2, 'VN'), 'EME', 'AE'))

#write_tsv(cpi_consolidated, 'cpi_consolidated.tsv')
```

Calculate the mean CPI for each group:

``` r
# Run
cpi_consolidated <- read_tsv('cpi_files/cpi_consolidated.tsv')
```

    ## Parsed with column specification:
    ## cols(
    ##   date = col_date(format = ""),
    ##   iso2c = col_character(),
    ##   value = col_double(),
    ##   classification = col_character()
    ## )

``` r
mean_groups <- cpi_consolidated %>% group_by(date, classification) %>% summarize(mean = mean(value, na.rm = T)) %>% ungroup
```

    ## `summarise()` regrouping by 'date' (override with `.groups` argument)

``` r
ggplot(cpi_consolidated, aes(x=date)) +
  geom_line(data = cpi_consolidated %>% filter(classification == 'EME'), aes(y=value, group = iso2c), color = 'salmon2', alpha = 0.7) +
  geom_line(data = cpi_consolidated %>% filter(classification == 'AE'), aes(y=value, group = iso2c), color = 'deepskyblue3', alpha = 0.7) +
  geom_line(data = (mean_groups %>% filter(classification == 'EME')), aes(x=date, y = mean, color = classification), size = 2) +
  geom_line(data = (mean_groups %>% filter(classification == 'AE')), aes(x=date, y = mean, color = classification), size = 2) +
  scale_color_manual("", labels = c("Average reserves' currencies", 'Average NIEs'), values = c('dodgerblue4', 'tomato3')) +
  scale_y_log10() +
  #annotate('text', x = as.Date('2018-03-01'), y = 133, label = 'CZ', size = 2.5) +
  #annotate('text', x = as.Date('2018-03-01'), y = 127, label = 'TH', size = 2.5) +
  #annotate('text', x = as.Date('2018-03-01'), y = 120, label = 'IL', size = 2.5) +
  #theme_bw(base_family = "MS Reference Sans Serif") +
  #ggthemes::theme_economist() +
  theme_bw() +
  theme(legend.position = c(0.25, 0.88),
        legend.background = element_blank()) +
  scale_x_date(breaks = '2 years', date_labels = '%Y') +
  labs(x='', y = 'Consumer price index (2005 = 100)')
```

![](figure2_cpi_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->
