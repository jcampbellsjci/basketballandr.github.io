Scraping Data
================
2017-3-23

You're interested in doing some data analysis, that's clear. Unfortunately you're missing something important... Data! There's plenty of sources of NBA data, and luckily for us, most of it's in a very structured format.

<!--more-->
In this post, I'm going to show you how to get data from a couple of different sources. The focus is going to be on scraping your own data from websites. You may be used to copying and pasting data into a spreadsheet or downloading it in `CSV` form. While this method is straightforward, when trying to collect large amounts of data it can be very cumbersome.

# Scraping a Basic HTML Table

The first example we're going to go over is scraping standings from Basketball-Reference. The standings are in a basic HTML format. We'll be using the `dplyr` and `rvest` packages throughout this post, so let's load them up now.

``` r
library(rvest)
library(dplyr)
```

`rvest` is used to scrape HTML data while `dplyr` is used to manipulate that pulled data and put it into a nice format.

Now we'll store the url we are going to be scraping from in an object called `bref`.

``` r
bref <- "https://www.basketball-reference.com/leagues/NBA_2020.html"
```

Now it's time to scrape some data. What you'll need is the CSS selector. The method for getting this can be different based on what browser you use. I use firefox, and I simply right click on the table and click on `inspect element`. From there, we can look for the line of code in the inspector that covers the table (the table will be highlighted once you hover over the code). Then, copy the selector; in this example it's `#confs_standings`.

Using `rvest`, we scrape the data and store it in `bref_standings`. We paste the css selector in the `read_html()` function. Because the data we are scraping is in table format, we use the command `html_table()`, which parses the table into a data frame.

``` r
bref_standings_list <- bref %>%
  read_html('#confs_standings') %>%
  html_table()
```

The data is stored in a list with four elements. The first two elements hold eastern and western conference standings. Let's pull the eastern conference out and take a look at it.

``` r
ec_standings <- bref_standings_list[[1]]

str(ec_standings)
```

    ## 'data.frame':    15 obs. of  8 variables:
    ##  $ Eastern Conference: chr  "Milwaukee Bucks*" "Toronto Raptors*" "Boston Celtics*" "Indiana Pacers*" ...
    ##  $ W                 : int  56 53 48 45 44 43 35 33 23 25 ...
    ##  $ L                 : int  17 19 24 28 29 30 37 40 42 47 ...
    ##  $ W/L%              : num  0.767 0.736 0.667 0.616 0.603 0.589 0.486 0.452 0.354 0.347 ...
    ##  $ GB                : chr  "—" "2.5" "7.5" "11.0" ...
    ##  $ PS/G              : num  119 113 114 109 112 ...
    ##  $ PA/G              : num  109 106 107 108 109 ...
    ##  $ SRS               : num  9.41 5.97 5.83 1.63 2.59 2.25 -1.01 -0.93 -7.03 -5.24 ...

We may want to clean this up a little. Two things we'll do is remove the standing placement next to the team names (we'll also rename this column to something more general) and fix this `GB` (games back) column to be numeric.

``` r
ec_standings <- ec_standings %>%
  # Rename Eastern Conference to TEAM
  rename(TEAM = `Eastern Conference`) %>%
  # Remove placement next to team name
  mutate(TEAM = trimws(gsub("[(].*", "", TEAM))) %>%
  # Editing GB
  mutate(GB = as.numeric(ifelse(GB == "-", NA, GB)))
```

Now we have a beautiful tibble to play around with!

# HTML Scraping Continued

Next, I want to go over an example that's a bit more involved. The focus on this section is more about dealing with basktball reference rather than the scraping itself, but it's a website you'll probably use a lot if you want to analyze NBA stats, so I think it's important to go over.

Basketball Reference makes their data easy to download, so if you just want one specific table it may be easier to just go to the website and download a `CSV`. However, we're going to go over an example which scrapes multiple tables.

The tables on Basketball Reference are in HTML format and some are easy to scrape; however, some tables are hidden inside comments, which make them difficult to deal with.

In this example we're going to scrape the Indiana Pacers' advanced stats for the past three seasons. An example of one of the tables we're gonna scrape is found [here](http://www.basketball-reference.com/teams/IND/2018.html). We're going to start by creating a list of urls for each of the three seasons. Then, we'll loop through them and paste each respective season into the Indiana Pacers' team url and store it in an object called `pacer_urls`.

``` r
library(purrr)

pacer_urls <- map(2018:2020,
                  ~ paste0("http://www.basketball-reference.com/teams/IND/",
                           .x, ".html"))
```

We're going to apply the same `rvest` methodology to scrape the advanced stats tables on each of the urls (the CSS selector in this case is `'advanced'`) and store the output in a list called `adv_pacers`. This will hold the advanced stats from each of the seasons we specified.

``` r
adv_pacers <- map(pacer_urls,
                  ~ read_html(.x) %>%
                    html_node('#advanced') %>%
                    html_table())
```

Looking at the first three rows of each data frame in the list, we can see we were successful!

``` r
map(adv_pacers, ~ head(.x, 3))
```

    ## [[1]]
    ##   Rk                  Age  G   MP  PER   TS%  3PAr   FTr ORB% DRB% TRB% AST%
    ## 1  1   Thaddeus Young  29 81 2607 14.8 0.528 0.209 0.106  8.0 14.1 11.1  8.5
    ## 2  2   Victor Oladipo  25 75 2552 23.1 0.577 0.323 0.274  2.1 15.1  8.6 21.2
    ## 3  3 Bojan Bogdanovic  28 80 2464 13.9 0.605 0.453 0.241  1.4 10.9  6.2  7.1
    ##   STL% BLK% TOV% USG%    OWS DWS  WS WS/48    OBPM DBPM  BPM VORP
    ## 1  2.6  1.2 10.4 17.3 NA 2.3 3.2 5.5 0.101 NA -0.2  0.4  0.2  1.4
    ## 2  3.5  2.0 12.7 30.1 NA 4.3 4.0 8.2 0.155 NA  4.1  1.7  5.8  5.0
    ## 3  1.1  0.3 10.1 19.0 NA 3.8 1.6 5.4 0.105 NA  0.5 -1.2 -0.7  0.8
    ## 
    ## [[2]]
    ##   Rk                  Age  G   MP  PER   TS%  3PAr   FTr ORB% DRB% TRB% AST%
    ## 1  1 Bojan Bogdanovic  29 81 2573 16.1 0.613 0.367 0.290  1.5 12.7  7.2  9.5
    ## 2  2   Thaddeus Young  30 81 2489 16.2 0.569 0.174 0.161  8.7 14.4 11.7 12.0
    ## 3  3  Darren Collison  31 76 2143 16.7 0.574 0.294 0.288  1.9  9.9  6.0 29.9
    ##   STL% BLK% TOV% USG%    OWS DWS  WS WS/48    OBPM DBPM BPM VORP
    ## 1  1.3  0.0 10.2 22.4 NA 3.9 2.8 6.8 0.126 NA  1.3 -0.6 0.7  1.7
    ## 2  2.4  1.3 12.0 18.0 NA 3.0 3.9 6.9 0.133 NA  0.4  1.1 1.5  2.2
    ## 3  2.5  0.4 14.4 17.7 NA 3.9 2.9 6.8 0.153 NA  0.8  1.0 1.8  2.1
    ## 
    ## [[3]]
    ##   Rk                  Age  G   MP  PER   TS%  3PAr   FTr ORB% DRB% TRB% AST%
    ## 1  1      T.J. Warren  26 67 2202 18.4 0.610 0.227 0.205  3.4 10.5  7.0  7.1
    ## 2  2 Domantas Sabonis  23 62 2159 20.7 0.586 0.079 0.349  9.7 29.3 19.6 21.7
    ## 3  3   Justin Holiday  30 73 1826 12.1 0.585 0.681 0.138  1.9 12.2  7.1  6.7
    ##   STL% BLK% TOV% USG%    OWS DWS  WS WS/48    OBPM DBPM BPM VORP
    ## 1  1.7  1.4  7.2 23.3 NA 4.0 2.6 6.5 0.143 NA  1.2  0.0 1.1  1.7
    ## 2  1.1  1.2 14.8 23.3 NA 4.1 3.4 7.5 0.168 NA  2.0  1.2 3.2  2.9
    ## 3  2.3  2.2  7.9 13.4 NA 1.6 2.6 4.2 0.112 NA -0.5  1.8 1.3  1.5

If you are scraping from basketball reference and are running into issues, it may be a result of parsing issues. If that's the case, try modifying your code to parse comments. If we ran into issues in the previous example, we could adjust our code like so:

``` r
map(pacer_urls,
    ~ read_html(.x) %>%
      # Code to parse the comments
      html_nodes(xpath = '//comment()') %>%
      html_text() %>%
      paste(collapse = '') %>%
      # Now go about the usual scraping
      read_html() %>%
      html_node('#advanced') %>%
      html_table())
```

# Scraping JSON Data

Another great source for NBA data is the NBA's own stats website. Getting data from the NBA site is a little different as it's stored in a JSON format rather than HTML. Don't let this scare you off, though, as getting data is just as easy!

There are multiple R libraries for scraping JSON data, but the one I'm most familiar with is `rjson`. We also need to use the `httr` package due to some additional discrepincies.

``` r
library(rjson)
library(httr)
```

We're going to scrape the traditional stats table for James Harden found at this [link](https://stats.nba.com/player/201935/). I am using firefox again in this instance, but chrome has a similar method of doing this.

First you need to call inspect element on the table, similar to what we did ealier. Next you're going to click on the `Network` header followed by the `XHR` tab. Now refresh the web page and you should be able to see several requests in the toolbar. You need to search around for the url we want, but you do have previews available (look at the previews by clicking on `response` in the tool bar with the options: headers, cookies, params, and response). After looking through a few of the requests, I found a url that has a parameter labeled PerGame. Since we want traditional per game stats, this sounds right. The url we want is listed under `Request URL` in the `Headers` section.

Let's try bringing the data into R. Recently, the NBA stats website made this process a little harder and require us to first specify some headers. You can use the following:

``` r
headers = c(
  `Connection` = 'keep-alive',
  `Accept` = 'application/json, text/plain, */*',
  `x-nba-stats-token` = 'true',
  `User-Agent` = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36',
  `x-nba-stats-origin` = 'stats',
  `Sec-Fetch-Site` = 'same-origin',
  `Sec-Fetch-Mode` = 'cors',
  `Referer` = 'http://stats.nba.com/%referer%/',
  `Accept-Encoding` = 'gzip, deflate, br',
  `Accept-Language` = 'en-US,en;q=0.9'
  )
```

First, we'll assign the JSON url to an object called `jh_url`. Than we will pull the raw data into an object called `jh_raw`.

``` r
jh_url <- "https://stats.nba.com/stats/playerdashboardbyyearoveryear?DateFrom=&DateTo=&GameSegment=&LastNGames=0&LeagueID=00&Location=&MeasureType=Base&Month=0&OpponentTeamID=0&Outcome=&PORound=0&PaceAdjust=N&PerMode=PerGame&Period=0&PlayerID=201935&PlusMinus=N&Rank=N&Season=2019-20&SeasonSegment=&SeasonType=Regular+Season&ShotClockRange=&Split=yoy&VsConference=&VsDivision="

jh_raw <- GET(jh_url, add_headers(.headers = headers))$content %>%
  rawToChar() %>%
  fromJSON()
```

There's a lot of info in this raw list. We could spend a while exploring, but here's how we can get the traditional stats we're looking for:

``` r
jh_df <- map_df(1:length(jh_raw$resultSets[[2]]$rowSet),
                ~ bind_cols(jh_raw$resultSets[[2]]$rowSet[[.x]]))

colnames(jh_df) <- jh_raw$resultSets[[2]]$headers
```

We use the `purrr` package to loop through elements of the list and assign them to a tibble using the `map_df` function. Here, we are focused on the `resultSets` portion of the `jh_raw` output. The second element of the `resultSets` portion is another list that stores each season of James Harden's stats in an element. We loop through that list (essentially looping through each season of Harden's career) and extract the output and combine it into a single tibble, where each row is a season of Harden's career.

Next, we assign the column names of this object to be the headers stored in this same second element of `resultSets`.

This is a pretty complicated example because the data is pretty messy and the NBA stats website has a ton of it (some of it helpful, others not). But here, we were able to get a clean tibble in only a few lines of code!

These few examples should be a good way to get your guys' hands on some data. Although these examples are related to a few NBA specific websites, they can easily be applied to websites with other information.
