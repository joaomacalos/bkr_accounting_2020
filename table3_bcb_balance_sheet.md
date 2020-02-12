Balance sheet of the BCB
================
João Pedro S. Macalós
2/10/2020

The objective of this notebook is to present the steps taken to scrape
the pdf balance sheets of the Brazilian Central Bank to build a table
with the values of its main items between 2005 and 2018. The software of
choice was Python due to the `camelot` library that is great to obtain
tables from pdf files.

The final table of this notebook is Table 3 from the paper “Does the
accounting framework affect the operational capacity of the central
bank? Lessons from the Brazilian experience” that is going to be
published on the Brazilian Keynesian Review.

This notebook was generated as a .Rmd file in Rstudio with the
`reticulate` package.

Import python libraries:

``` python
import pandas as pd
import camelot
import numpy as np
import matplotlib.pyplot as plt
import os
import glob
```

``` python
files = sorted(glob.glob('balance_sheets_pdf/*-02.pdf'))
```

``` python
tables = []

for f in files:
    t = camelot.read_pdf(f, pages = '2', flavor='stream', edge_tol = 500)
    tables.append(t)
```

``` python
tables[0][0].df.head()
```

    ##                                        0      1  ...     6              7
    ## 0                BANCO CENTRAL DO BRASIL         ...                     
    ## 1  BALANÇO PATRIMONIAL EM 31 DE DEZEMBRO         ...                fl. 1
    ## 2                   Em milhares de Reais         ...                     
    ## 3                              A T I V O  Notas  ...  2005           2004
    ## 4                                                ...        (Republicado)
    ## 
    ## [5 rows x 8 columns]

## Assets

We must define some functions and patterns that are going to be useful
to clean the PDF files and select the data:

``` python
pattern = ('EXTERNO|' + 
           'ATIVO EM MOEDAS ESTRANGEIRAS|' + 
           'INTERNO|' + 
           'ATIVO EM MOEDA LOCAL|' + 
           'Empréstimos|' + 
           'Títulos Públicos|' + 
           'Compromisso de Revenda|' + 
           'Outros Créditos|' + 
           'TOTAL')
```

``` python
def clean_assets(df):
    t = df[df.vars.str.contains(pattern)]
    t = (t.
        assign(vars = t.vars.str.replace('(Nota [0-9].*)',  '').
            str.replace('(', '').str.replace(')', '').
            str.replace('-', '')))
    t = t.reset_index(drop=True)
    return(t)
```

``` python
def clean_assets2(df, year, include = False):
    t = df[0].df.loc[:, :2].drop(1, axis = 1)
        
    t.columns = ['vars', year]
    t = clean_assets(t)
    return(t)
```

``` python
def clean_names(df):
    t = df.assign(vars = (
                  pd.np.where(df.vars.str.contains("ESTRAN|EXTERNO"), "EXTERNO",
                  pd.np.where(df.vars.str.contains("LOCAL|INTERNO"), "INTERNO",
                  pd.np.where(df.vars.str.contains("Títulos Públicos"), "GOVT_SECS",
                  pd.np.where(df.vars.str.contains('Empréstimos'), 'LOANS_TO_BANKS',
                  pd.np.where(df.vars.str.contains('Outros'), 'OUTROS',
                  pd.np.where(df.vars.str.contains('TOTAL'), 'TOTAL',
                  pd.np.where(df.vars.str.contains('Revenda'),"REPOS", df.vars)))))))))
    return(t)
```

Get the relevant rows for 2005:

``` python
bsa_2005 = clean_assets2(tables[0], '2005')
bsa_2005 = clean_names(bsa_2005).drop(1)
bsa_2005
```

    ##         vars         2005
    ## 0    EXTERNO  140.474.794
    ## 2    INTERNO  343.217.073
    ## 3      REPOS   25.941.192
    ## 4  GOVT_SECS  281.393.821
    ## 5      TOTAL  483.691.867

Create a list of the remaining years (2006 to 2018):

``` python
yrs = list(range(2006, 2019))
yrs = list(map(lambda x: str(x), yrs))
```

Join all tables together

``` python
assets = bsa_2005

for table, year in zip(tables[1:14], yrs):
    step1 = clean_assets2(table, year)
    step1 = clean_names(step1).drop(1)
    assets = assets.merge(step1, on = 'vars', how = 'left')
    
assets
```

    ##         vars         2005  ...           2017           2018
    ## 0    EXTERNO  140.474.794  ...  1.363.766.435  1.601.808.345
    ## 1    INTERNO  343.217.073  ...  1.812.230.232  1.878.538.055
    ## 2      REPOS   25.941.192  ...            NaN         14.040
    ## 3  GOVT_SECS  281.393.821  ...  1.662.315.859  1.795.199.557
    ## 4      TOTAL  483.691.867  ...  3.175.996.667  3.480.346.400
    ## 
    ## [5 rows x 15 columns]

Reshape to long format:

``` python
assets2 = (assets.
          apply(lambda x: x.replace('-', np.nan)).fillna(0).
          melt(id_vars = ['vars'], var_name = 'date', value_name = 'value').
          pivot(index = 'date', columns = 'vars', values = 'value').
          apply(lambda x: x.replace('\.', '', regex = True)).
          apply(lambda x: pd.to_numeric(x, errors = 'coerce'))
          )
          
assets2
```

    ## vars     EXTERNO   GOVT_SECS     INTERNO     REPOS       TOTAL
    ## date                                                          
    ## 2005   140474794   281393821   343217073  25941192   483691867
    ## 2006   200980845   303860298   343448875    504501   544429720
    ## 2007   358117237   359335362   408234298   2790896   766351535
    ## 2008   512512891   496741066   534579563     44298  1047092454
    ## 2009   429635304   640215918   727960902         0  1157596206
    ## 2010   496109813   703175643   794189768         0  1290299581
    ## 2011   675500413   754543113   907911058   9299998  1583411471
    ## 2012   784189650   910222934  1024758273  61849997  1808947923
    ## 2013   900658954   953068070  1007026968      5403  1907685922
    ## 2014  1008907527  1113234371  1148122839         0  2157030366
    ## 2015  1471172680  1279138194  1312701235         0  2783873915
    ## 2016  1292650832  1518007723  1739477604         0  3032128436
    ## 2017  1363766435  1662315859  1812230232         0  3175996667
    ## 2018  1601808345  1795199557  1878538055     14040  3480346400

## Liabilities

To get the liabilities, the logic of the codes and functions are similar
to the ones used to find the assets but adapted to the liabilities of
the BCB:

``` python
pattern_liab = (#'EXTERNO|' + 
                #'PASSIVO EM MOEDAS ESTRANGEIRAS|' + 
                'INTERNO|' + 
                'PASSIVO EM MOEDA LOCAL|' + 
                'Depósitos de Instituições|' + 
                #'Notas do Banco|' + 
                'a Ordem do Governo|à Ordem do Governo|Obrigações com o Governo|' + 
                'Compromisso de Recompra|Compromissos de Recompra|' + 
                'MEIO CIRCULANTE|' + 
                'PATRIMÔNIO|' + 
                'TOTAL')
```

``` python
def clean_liab(df):
    t = df[df.vars.str.contains(pattern_liab)]
    t = (t. 
        assign(vars = t.vars.str.replace('(Nota [0-9].*)',  '').
        str.replace('(', '').str.replace(')', '').
        str.replace('-', ''))
        )
    t = t.reset_index(drop=True)
    return(t)
```

``` python
def clean_liab2(df, year, o3 = False):
    if o3 is True:
        t = df[0].df.loc[:, 5:7].drop(6, axis = 1)
    else:
        t = df[0].df.loc[:, 4:6].drop(5, axis = 1)
        
    t.columns = ['vars', year]
    t = clean_liab(t)
    return(t)
```

``` python
def clean_names_liab(df):
    t = df.assign(vars = (
                  #pd.np.where(df.vars.str.contains("ESTRAN|EXTERNO"), "EXTERNO",
                  pd.np.where(df.vars.str.contains("LOCAL|INTERNO"), "INTERNO",
                  pd.np.where(df.vars.str.contains('Depósitos de Instituições'), 'BANK_DEPS',
                  #pd.np.where(df.vars.str.contains("Reservas Bancárias"), "RESERVES",
                  pd.np.where(df.vars.str.contains('Governo Federal'), 'GOVT_DEPS',
                  #pd.np.where(df.vars.str.contains('Notas'), 'CB_BONDS',
                  pd.np.where(df.vars.str.contains('Recompra'), 'REPOS',
                  pd.np.where(df.vars.str.contains('MEIO'), 'BANKNOTES',
                  pd.np.where(df.vars.str.contains('TOTAL'), 'TOTAL',
                  pd.np.where(df.vars.str.contains('PATRIMÔNIO'),"CAPITAL", df.vars))))))))#)))
                  )
    idx = t.index[t['vars'].str.contains('INTERNO')].tolist()
    t = t.iloc[idx[0]+1:,:]
    return(t)
```

``` python
bsp_2005 = clean_liab2(tables[0], '2005')
bsp_2005 = clean_names_liab(bsp_2005)
bsp_2005
```

    ##         vars         2005
    ## 2  BANK_DEPS  104.545.368
    ## 3      REPOS   63.109.520
    ## 4  GOVT_DEPS  210.676.399
    ## 5  BANKNOTES   70.033.641
    ## 6    CAPITAL    8.803.966
    ## 7      TOTAL  483.691.867

From 2005 to 2008:

``` python
liabilities = bsp_2005

for table, year in zip(tables[1:4], yrs[0:3]):
    step1 = clean_liab2(table, year)
    step1 = clean_names_liab(step1)
    liabilities = liabilities.merge(step1, on = 'vars', how = 'left')
    
liabilities
```

    ##         vars         2005         2006         2007           2008
    ## 0  BANK_DEPS  104.545.368  118.438.655  145.973.427     90.035.395
    ## 1      REPOS   63.109.520   77.871.622  190.207.090    345.735.757
    ## 2  GOVT_DEPS  210.676.399  226.456.810  276.333.619    437.426.384
    ## 3  BANKNOTES   70.033.641   85.824.753  102.885.047    115.590.704
    ## 4    CAPITAL    8.803.966   13.720.738    1.006.654     14.227.611
    ## 5      TOTAL  483.691.867  544.429.720  766.351.535  1.047.092.454

2009 is special since it has an extra column:

``` python
bsp_2009 = clean_liab2(tables[4], '2009', o3 = True)
bsp_2009 = clean_names_liab(bsp_2009)
liabilities = liabilities.merge(bsp_2009, on = 'vars', how = 'left')
liabilities
```

    ##         vars         2005  ...           2008           2009
    ## 0  BANK_DEPS  104.545.368  ...     90.035.395     97.077.510
    ## 1      REPOS   63.109.520  ...    345.735.757    454.709.678
    ## 2  GOVT_DEPS  210.676.399  ...    437.426.384    413.807.893
    ## 3  BANKNOTES   70.033.641  ...    115.590.704    131.861.185
    ## 4    CAPITAL    8.803.966  ...     14.227.611     20.098.650
    ## 5      TOTAL  483.691.867  ...  1.047.092.454  1.157.596.206
    ## 
    ## [6 rows x 6 columns]

``` python
for table, year in zip(tables[5:8], yrs[4:7]):
    step1 = clean_liab2(table, year)
    step1 = clean_names_liab(step1)
    liabilities = liabilities.merge(step1, on = 'vars', how = 'left')
    
liabilities
```

    ##         vars         2005  ...           2011           2012
    ## 0  BANK_DEPS  104.545.368  ...    424.925.295    320.097.305
    ## 1      REPOS   63.109.520  ...    351.178.116    597.214.923
    ## 2  GOVT_DEPS  210.676.399  ...    578.190.914    633.537.608
    ## 3  BANKNOTES   70.033.641  ...    162.769.670    187.434.736
    ## 4    CAPITAL    8.803.966  ...     18.830.516     21.524.159
    ## 5      TOTAL  483.691.867  ...  1.583.411.471  1.808.947.923
    ## 
    ## [6 rows x 9 columns]

``` python
bsp_2013 = clean_liab2(tables[8], '2013', o3 = True)
bsp_2013 = clean_names_liab(bsp_2013)
liabilities = liabilities.merge(bsp_2013, on = 'vars', how = 'left')
liabilities
```

    ##         vars         2005  ...           2012           2013
    ## 0  BANK_DEPS  104.545.368  ...    320.097.305    369.095.050
    ## 1      REPOS   63.109.520  ...    597.214.923    568.885.481
    ## 2  GOVT_DEPS  210.676.399  ...    633.537.608    687.081.449
    ## 3  BANKNOTES   70.033.641  ...    187.434.736    204.052.420
    ## 4    CAPITAL    8.803.966  ...     21.524.159     18.596.394
    ## 5      TOTAL  483.691.867  ...  1.808.947.923  1.907.685.922
    ## 
    ## [6 rows x 10 columns]

``` python
for table, year in zip(tables[9:], yrs[8:]):
    step1 = clean_liab2(table, year)
    step1 = clean_names_liab(step1)
    liabilities = liabilities.merge(step1, on = 'vars', how = 'left')
    
liabilities
```

    ##         vars         2005  ...           2017           2018
    ## 0  BANK_DEPS  104.545.368  ...    453.729.168    444.152.075
    ## 1      REPOS   63.109.520  ...  1.091.328.757  1.175.999.993
    ## 2  GOVT_DEPS  210.676.399  ...  1.095.957.988  1.302.160.762
    ## 3  BANKNOTES   70.033.641  ...    250.363.681    264.967.669
    ## 4    CAPITAL    8.803.966  ...    124.243.379    126.665.323
    ## 5      TOTAL  483.691.867  ...  3.175.996.667  3.480.346.400
    ## 
    ## [6 rows x 15 columns]

``` python
liab2 = (liabilities.
        apply(lambda x: x.replace('\.', '', regex = True)).
        fillna(0).
        melt(id_vars = ['vars'], var_name = 'date', value_name = 'value').
        pivot(index = 'date', columns = 'vars', values = 'value').
        apply(lambda x: pd.to_numeric(x, errors = 'coerce'))
        )
        
liab2
```

    ## vars  BANKNOTES  BANK_DEPS    CAPITAL   GOVT_DEPS       REPOS       TOTAL
    ## date                                                                     
    ## 2005   70033641  104545368    8803966   210676399    63109520   483691867
    ## 2006   85824753  118438655   13720738   226456810    77871622   544429720
    ## 2007  102885047  145973427    1006654   276333619   190207090   766351535
    ## 2008  115590704   90035395   14227611   437426384   345735757  1047092454
    ## 2009  131861185   97077510   20098650   413807893   454709678  1157596206
    ## 2010  151145368  379441614   15958637   410521771   288665899  1290299581
    ## 2011  162769670  424925295   18830516   578190914   351178116  1583411471
    ## 2012  187434736  320097305   21524159   633537608   597214923  1808947923
    ## 2013  204052420  369095050   18596394   687081449   568885481  1907685922
    ## 2014  220853706  325872059   18710015   697896062   837124219  2157030366
    ## 2015  225485184  368414269  103481550  1036601593   967748493  2783873915
    ## 2016  232145593  409224031  125816034  1050206705  1085349829  3032128436
    ## 2017  250363681  453729168  124243379  1095957988  1091328757  3175996667
    ## 2018  264967669  444152075  126665323  1302160762  1175999993  3480346400

## Combine assets and liabilities and normalize to Brazilian GDP

Download Brazilian GDP (accumulated in 12 monhts) using `sgs` library:

``` python
import sgs
```

``` python
gdp_br = sgs.time_serie(4382, start = '01/01/2005', end = '31/12/2018')
```

``` python
gdp_br = pd.DataFrame(gdp_br)
gdp_br1 = (gdp_br.assign(month = lambda x: x.index.month).
          query('month == 12').
          rename(columns = {4382:'gdp'}).
          assign(date = lambda x: x.index.year)
          )
          
gdp_br1 = gdp_br1.reset_index()[['date', 'gdp']]

gdp_br1.head()
```

    ##    date        gdp
    ## 0  2005  2170584.5
    ## 1  2006  2409449.9
    ## 2  2007  2720262.9
    ## 3  2008  3109803.1
    ## 4  2009  3333039.4

Normalize assets as a ratio of
GDP:

``` python
assets3 = assets2.reset_index().assign(date = lambda x: x.date.astype('int64'))

assets3 = gdp_br1.merge(assets3, how = 'left', on = 'date').iloc[:, :4]

assets3.columns = ['date', 'gdp', 'a_ext', 'a_gs']

assets4 = (assets3.assign(a_ext = lambda x: 100 * x.a_ext / (1000 * x.gdp))
        .assign(a_gs = lambda x: 100 * x.a_gs / (1000 * x.gdp))
        .drop('gdp', 1)
        )

assets4.head()
```

    ##    date      a_ext       a_gs
    ## 0  2005   6.471750  12.963965
    ## 1  2006   8.341358  12.611190
    ## 2  2007  13.164802  13.209582
    ## 3  2008  16.480558  15.973393
    ## 4  2009  12.890196  19.208171

Normalize liabilites as a ratio of
GDP:

``` python
liab3 = liab2.reset_index().assign(date = lambda x: x.date.astype('int64'))

liab4 = gdp_br1.merge(liab3, how = 'left', on = 'date').drop(columns = ['TOTAL'])
liab4.columns = ['date', 'gdp', 'l_bn', 'l_bkres', 'l_cap', 'l_gd', 'l_repos']

cols_to_norm = ['l_bn', 'l_bkres', 'l_cap', 'l_gd', 'l_repos']

liab4[[col for col in cols_to_norm]] = 100 * liab4[cols_to_norm].div(1000 * liab4.gdp, axis = 'index')

liab4 = liab4.drop(columns = ['gdp'])
liab4
```

    ##     date      l_bn   l_bkres     l_cap       l_gd    l_repos
    ## 0   2005  3.226488  4.816462  0.405603   9.705975   2.907490
    ## 1   2006  3.562006  4.915589  0.569455   9.398693   3.231925
    ## 2   2007  3.782173  5.366151  0.037006  10.158342   6.992232
    ## 3   2008  3.716978  2.895212  0.457508  14.066048  11.117609
    ## 4   2009  3.956184  2.912582  0.603013  12.415332  13.642493
    ## 5   2010  3.889638  9.764708  0.410686  10.564538   7.428648
    ## 6   2011  3.719275  9.709511  0.430276  13.211619   8.024394
    ## 7   2012  3.892920  6.648250  0.447045  13.158239  12.403836
    ## 8   2013  3.827213  6.922757  0.348795  12.886920  10.670033
    ## 9   2014  3.821691  5.638946  0.323761  12.076514  14.485742
    ## 10  2015  3.760727  6.144552  1.725904  17.288833  16.140475
    ## 11  2016  3.702878  6.527399  2.006850  16.751504  17.312060
    ## 12  2017  3.803001  6.892104  1.887245  16.647499  16.577182
    ## 13  2018  3.846145  6.447100  1.838614  18.901546  17.070256

Consolidate BCB balance sheet:

``` python
bcb_bsheet = (assets4.merge(liab4, on = 'date', how = 'left')
             .assign(date = lambda x: x.date.astype('str')))
             
bcb_bsheet.head()
```

    ##    date      a_ext       a_gs  ...     l_cap       l_gd    l_repos
    ## 0  2005   6.471750  12.963965  ...  0.405603   9.705975   2.907490
    ## 1  2006   8.341358  12.611190  ...  0.569455   9.398693   3.231925
    ## 2  2007  13.164802  13.209582  ...  0.037006  10.158342   6.992232
    ## 3  2008  16.480558  15.973393  ...  0.457508  14.066048  11.117609
    ## 4  2009  12.890196  19.208171  ...  0.603013  12.415332  13.642493
    ## 
    ## [5 rows x 8 columns]

Convert to R to
    present:

``` r
library(tidyverse)
```

    ## ── Attaching packages ───────────────────────────────────────────────────────────────────────────────── tidyverse 1.3.0 ──

    ## ✓ ggplot2 3.2.1     ✓ purrr   0.3.3
    ## ✓ tibble  2.1.3     ✓ dplyr   0.8.3
    ## ✓ tidyr   1.0.0     ✓ stringr 1.4.0
    ## ✓ readr   1.3.1     ✓ forcats 0.4.0

    ## ── Conflicts ──────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(reticulate)
```

``` r
bcb_bs <- py$bcb_bsheet %>%
  mutate_at(vars(-1), list(~ round(., digits = 1))) %>%
  #as_tibble(rownames = 'year') %>%
  mutate(Assets = '',
         Liabilities = '') %>%
  select(1, 9, 2, 3, 10, 5, 8, 7, 4, 6) %>%
  set_names('rows/date', 'Assets', "Foreign Assets", 'Govt. Securities', 
            'Liabilities', 'Bank reserves', 'Rev. Repos', 'Govt. Deposits',
            'Currency in circulation', 'Equity') %>% 
  t() %>%
  as_tibble(rownames = '.') %>%
  set_names(.[1,]) %>%
  slice(-1)
```

``` r
library(kableExtra)
```

    ## 
    ## Attaching package: 'kableExtra'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     group_rows

``` r
options("kableExtra.html.bsTable" = T)
kable(bcb_bs, format = 'markdown') %>%
 kable_styling(bootstrap_options = c("striped", "hover"),
               font_size = 9,
               full_width = F)
```

| rows/date               | 2005 | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 | 2014 | 2015 | 2016 | 2017 | 2018 |
| :---------------------- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Assets                  |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| Foreign Assets          | 6.5  | 8.3  | 13.2 | 16.5 | 12.9 | 12.8 | 15.4 | 16.3 | 16.9 | 17.5 | 24.5 | 20.6 | 20.7 | 23.3 |
| Govt. Securities        | 13.0 | 12.6 | 13.2 | 16.0 | 19.2 | 18.1 | 17.2 | 18.9 | 17.9 | 19.3 | 21.3 | 24.2 | 25.3 | 26.1 |
| Liabilities             |      |      |      |      |      |      |      |      |      |      |      |      |      |      |
| Bank reserves           | 4.8  | 4.9  | 5.4  | 2.9  | 2.9  | 9.8  | 9.7  | 6.6  | 6.9  | 5.6  | 6.1  | 6.5  | 6.9  | 6.4  |
| Rev. Repos              | 2.9  | 3.2  | 7.0  | 11.1 | 13.6 | 7.4  | 8.0  | 12.4 | 10.7 | 14.5 | 16.1 | 17.3 | 16.6 | 17.1 |
| Govt. Deposits          | 9.7  | 9.4  | 10.2 | 14.1 | 12.4 | 10.6 | 13.2 | 13.2 | 12.9 | 12.1 | 17.3 | 16.8 | 16.6 | 18.9 |
| Currency in circulation | 3.2  | 3.6  | 3.8  | 3.7  | 4.0  | 3.9  | 3.7  | 3.9  | 3.8  | 3.8  | 3.8  | 3.7  | 3.8  | 3.8  |
| Equity                  | 0.4  | 0.6  | 0.0  | 0.5  | 0.6  | 0.4  | 0.4  | 0.4  | 0.3  | 0.3  | 1.7  | 2.0  | 1.9  | 1.8  |

``` r
#bcb_bs
```
