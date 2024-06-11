
# Wordmap: multi-purpose lexicon generator

<!-- badges: start -->

[![CRAN
Version](https://www.r-pkg.org/badges/version/wordmap)](https://CRAN.R-project.org/package=wordmap)
[![Downloads](https://cranlogs.r-pkg.org/badges/wordmap)](https://CRAN.R-project.org/package=wordmap)
[![Total
Downloads](https://cranlogs.r-pkg.org/badges/grand-total/wordmap?color=orange)](https://CRAN.R-project.org/package=wordmap)
[![R build
status](https://github.com/koheiw/wordmap/workflows/R-CMD-check/badge.svg)](https://github.com/koheiw/wordmap/actions)
[![codecov](https://codecov.io/gh/koheiw/wordmap/branch/master/graph/badge.svg)](https://codecov.io/gh/koheiw/wordmap)
<!-- badges: end -->

## How to install

**wordmap** is available on CRAN since the version 0.6. You can install
the package using R Studio GUI or the command.

``` r
install.packages("wordmap")
```

If you want to the latest version, please install by running this
command in R. You need to have **devtools** installed beforehand.

``` r
install.packages("devtools")
devtools::install_github("koheiw/wordmap")
```

## Example

In this example, using a text analysis package
[**quanteda**](https://quanteda.io) for preprocessing of textual data,
we train a geographical classification model on a [corpus of news
summaries collected from Yahoo
News](https://www.dropbox.com/s/e19kslwhuu9yc2z/yahoo-news.RDS?dl=1) via
RSS in 2014.

### Download example data

``` r
download.file('https://www.dropbox.com/s/e19kslwhuu9yc2z/yahoo-news.RDS?dl=1', 
              '~/yahoo-news.RDS', mode = "wb")
```

### Train wordmap classifier

``` r
require(wordmap)
## Loading required package: wordmap
```

``` r
require(quanteda)
## Loading required package: quanteda
## Package version: 4.0.2
## Unicode version: 15.1
## ICU version: 74.1
## Parallel computing: 16 of 16 threads used.
## See https://quanteda.io for tutorials and examples.
```

``` r

# Load data
dat <- readRDS('~/yahoo-news.RDS')
dat$text <- paste0(dat$head, ". ", dat$body)
dat$body <- NULL
corp <- corpus(dat, text_field = 'text')

# Custom stopwords
month <- c('January', 'February', 'March', 'April', 'May', 'June',
           'July', 'August', 'September', 'October', 'November', 'December')
day <- c('Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday')
agency <- c('AP', 'AFP', 'Reuters')

# Select training period
sub_corp <- corpus_subset(corp, '2014-01-01' <= date & date <= '2014-12-31')

# Tokenize
toks <- tokens(sub_corp)
toks <- tokens_remove(toks, stopwords('english'), valuetype = 'fixed', padding = TRUE)
toks <- tokens_remove(toks, c(month, day, agency), valuetype = 'fixed', padding = TRUE)

# quanteda v1.5 introduced 'nested_scope' to reduce ambiguity in dictionary lookup
toks_label <- tokens_lookup(toks, data_dictionary_wordmap_en, 
                            levels = 3, nested_scope = "dictionary")
dfmt_label <- dfm(toks_label)

dfmt_feat <- dfm(toks, tolower = FALSE)
dfmt_feat <- dfm_select(dfmt_feat, selection = "keep", '^[A-Z][A-Za-z1-2]+', 
                        valuetype = 'regex', case_insensitive = FALSE) # include only proper nouns to model
dfmt_feat <- dfm_trim(dfmt_feat, min_termfreq = 10)

model <- textmodel_wordmap(dfmt_feat, dfmt_label)

# Features with largest weights
coef(model, n = 7)[c("us", "gb", "fr", "br", "jp")]
## $us
## WASHINGTON Washington         US  Americans       YORK     States        NYC 
##  10.032258   9.497570   8.788279   8.235422   6.951923   6.286159   6.124962 
## 
## $gb
##    London    LONDON   Britain Britain's        UK   British    Briton 
## 10.654615 10.648236 10.397470  9.751142  9.711813  7.846246  7.534446 
## 
## $fr
##     France      PARIS      Paris     French      Valls Hollande's  Frenchman 
##  11.322985  10.449373  10.260425   8.166990   8.005540   7.970754   7.838486 
## 
## $br
##    Brazil Brazilian       SAO     PAULO       RIO   JANEIRO       Rio 
##  11.63401  10.33473  10.28710  10.28257  10.21209  10.20525  10.09771 
## 
## $jp
##     Japan  Japanese     TOKYO     Tokyo       Abe     Abe's    Shinzo 
## 11.751948 10.938451 10.795585 10.101430  8.653604  8.065388  7.983628
```

### Predict geographical focus of texts

``` r
pred_data <- data.frame(text = as.character(sub_corp), country = predict(model))
```

|        | text                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | country |
|:-------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------|
| text63 | ’08 French champ Ivanovic loses to Safarova in 3rd. PARIS (AP) — Former French Open champion Ana Ivanovic lost in the third round Saturday, beaten 6-3, 6-3 by 23rd-seeded Lucie Safarova of the Czech Republic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | fr      |
| text68 | Up to \$1,000 a day to care for child migrants. More than 57,000 unaccompanied children, mostly from Central America, have been caught entering the country illegally since last October, and President Barack Obama has asked for \$3.7 billion in emergency funding to address what he has called an “urgent humanitarian solution.” “One of the figures that sticks in everybody’s mind is we’re paying about \$250 to \$1,000 per child,” Senator Jeff Flake told reporters, citing figures presented at a closed-door briefing by Homeland Security Secretary Jeh Johnson. Federal authorities are struggling to find more cost-effective housing, medical care, counseling and legal services for the undocumented minors. The base cost per bed was \$250 per day, including other services, Senator Dianne Feinstein said, without providing details. | us      |
| text69 | 1,000 DRC ex-rebels break out of Uganda camp: army. About 1,000 former fighters from a former Democratic Republic of Congo rebel group broke out Tuesday from a camp where they being held in Uganda just as soldiers were about to repatriate them, the Ugandan army said. “A thousand rebels from the M23 (group) have escaped” from the camp in Bihanga, about 300 kilometres (190 miles) southwest of the Ugandan capital Kampala, a spokesman for the Ugandan army said on the official Twitter account.                                                                                                                                                                                                                                                                                                                                                 | ug      |
| text73 | 1,000 killed in Boko Haram conflict this year. More than 1,000 people have been killed so far this year in three states in northeastern Nigeria worst hit by Boko Haram violence, according to the country’s main relief organisation. The National Emergency Management Agency (NEMA) figures are the starkest indication yet of the increase in bloodshed in Borno, Adamawa and Yobe that have caused growing concern. NEMA said in a presentation in Abuja on Tuesday that people living in the states were “caught up in an intensifying conflict”, which has been raging since 2009. Violence has increased in northeastern Nigeria since the new year, including a high-profile attack on a boarding school in Yobe, which saw dozens of students slaughtered in their beds.                                                                            | ng      |
| text78 | 1,000 migrants repulsed at Spanish border. MADRID (AP) — Officials say around 1,000 migrants of sub-Saharan origin have failed in an attempt to get over Spain’s three-tier barbed-wire border fence separating its North African enclave of Melilla from Morocco in a bid to enter Europe.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | es      |
| text79 | Some 1,000 migrants try to reach Spain from Africa. MADRID (AP) — Some 700 migrants stormed a border fence to try to enter Spain’s northwest African enclave city of Melilla from Morocco on Tuesday while the sea rescue service said it had picked up some 500 others trying to enter the country clandestinely by crossing the Strait of Gibraltar in boats, officials said.                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | es      |
