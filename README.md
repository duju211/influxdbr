
<!-- README.md is generated from README.Rmd. Please edit that file -->
influxdbr
=========

[![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version/influxdbr)](https://cran.r-project.org/package=influxdbr) [![Build Status](https://travis-ci.org/dleutnant/influxdbr.svg?branch=master)](https://travis-ci.org/dleutnant/influxdbr)

R interface to [InfluxDB](https://docs.influxdata.com/influxdb)

This package allows you to fetch and write time series data from/to an InfluxDB server. Additionally, handy wrappers for the Influx Query Language (IQL) to manage and explore a remote database are provided.

Installation
------------

Installation is easy thanks to CRAN:

``` r
install.packages("influxdbr")
```

You can install the dev version influxdbr from github with:

``` r
# install.packages("devtools")
devtools::install_github("dleutnant/influxdbr")
```

Example
-------

This is a basic example which shows you how to communicate (i.e. query and write data) with the InfluxDB server.

``` r
library(xts)
#> Loading required package: zoo
#> 
#> Attaching package: 'zoo'
#> The following objects are masked from 'package:base':
#> 
#>     as.Date, as.Date.numeric
library(influxdbr)
```

Let's create first some sample data from the xts package and assign arbitrary attributes:

``` r
# attach data "sample_matrix"
data("sample_matrix")

# create xts object
xts_data <- xts::as.xts(x = sample_matrix)

# assign some attributes
xts::xtsAttributes(xts_data) <- list(info = "SampleDataMatrix",
                                     UnitTesting = TRUE, 
                                     n = 180)
                                     
# print structure to inspect the object
str(xts_data)
#> An 'xts' object on 2007-01-02/2007-06-30 containing:
#>   Data: num [1:180, 1:4] 50 50.2 50.4 50.4 50.2 ...
#>  - attr(*, "dimnames")=List of 2
#>   ..$ : NULL
#>   ..$ : chr [1:4] "Open" "High" "Low" "Close"
#>   Indexed by objects of class: [POSIXct,POSIXt] TZ: 
#>   xts Attributes:  
#> List of 3
#>  $ info       : chr "SampleDataMatrix"
#>  $ UnitTesting: logi TRUE
#>  $ n          : num 180
```

### InfluxDB connection

To connect to an InfluxDB server, we need a connection object. A connection object can be created by providing usual server details (e.g. `host`, `port`, ...) or with help of a group file, which conveniently holds all information for us (s. package documentation):

``` r
# create connection object 
# (here: based on a config file with group "admin" in it (s. package documentation))
con <- influx_connection(group = "admin")
#> Success: (204) No Content
```

The `influxdbr` package provides handy wrappers to manage a remote InfluxDB:

``` r
# create new database
create_database(con = con, db = "mydb")

# list all databases
show_databases(con = con)
#> # A tibble: 12 x 1
#>         name
#>        <chr>
#>  1 _internal
#>  2    stbmod
#>  3     wasig
#>  4  wasig-fr
#>  5   wasig-h
#>  6      tmp2
#>  7      tmp3
#>  8      test
#>  9    new_db
#> 10      mydb
#> 11    new_df
#> 12       tmp
```

### Write data

Writing an xts-object to the server can be achieved with `influx_write`. In this case, columnnames of the `xts` object are used as InfluxDB's field keys, `xts`'s coredata represent field values. Attributes are preserved and written as tag keys and values, respectively.

``` r
# write example xts-object to database
influx_write(con = con, 
             db = "mydb",
             x = xts_data, 
             measurement = "sampledata")
```

We can now check if the time series was succefully written:

``` r
# check if measurement was succefully written
show_measurements(con = con, db = "mydb")
#> # A tibble: 1 x 1
#>         name
#>        <chr>
#> 1 sampledata
```

### Query data

To query the database, two functions `influx_query` and `influx_select` are available. `influx_select` wraps around `influx_query` and can be useful for simple requests because it provides default query parameters. The return type can be configured to be of class `tibble` or of class `xts`.

#### Return tibbles

ToDo: List of tibbles, each with statement\_id, series\_names, tags, time, fields

``` r
# fetch time series data by using the helper function `influx_select`
result <- influx_select(con = con, 
                        db = "mydb", 
                        field_keys = "Open, High", 
                        measurement = "sampledata",
                        group_by = "*",
                        limit = 10, 
                        order_desc = TRUE, 
                        return_xts = FALSE)

result
#> [[1]]
#> # A tibble: 10 x 9
#>    statement_id series_names series_partial UnitTesting             info
#>           <int>        <chr>          <lgl>       <chr>            <chr>
#>  1            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  2            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  3            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  4            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  5            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  6            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  7            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  8            0   sampledata          FALSE        TRUE SampleDataMatrix
#>  9            0   sampledata          FALSE        TRUE SampleDataMatrix
#> 10            0   sampledata          FALSE        TRUE SampleDataMatrix
#> # ... with 4 more variables: n <chr>, time <dttm>, Open <dbl>, High <dbl>
```

#### Return xts

ToDo: List of xts. Because xts objects are basically matrices (which can store one data type only), a single xts object is created for each InfluxDB field. This ensures a correct representation of the field values data type (instead of getting all into a "character" matrix). InfluxDB tags are now xts attributes.

``` r
# fetch time series data by using the helper function `influx_select`
result <- influx_select(con = con, 
                        db = "mydb", 
                        field_keys = "Open, High", 
                        measurement = "sampledata",
                        group_by =  "*",
                        limit = 10, 
                        order_desc = TRUE, 
                        return_xts = TRUE)

str(result)
#> List of 1
#>  $ :List of 2
#>   ..$ sampledata:An 'xts' object on 2007-06-20 22:00:00/2007-06-29 22:00:00 containing:
#>   Data: num [1:10, 1] 47.7 47.6 47.2 47.2 47.2 ...
#>  - attr(*, "dimnames")=List of 2
#>   ..$ : NULL
#>   ..$ : chr "Open"
#>   Indexed by objects of class: [POSIXct,POSIXt] TZ: GMT
#>   xts Attributes:  
#> List of 6
#>   .. ..$ statement_id  : int 0
#>   .. ..$ series_names  : chr "sampledata"
#>   .. ..$ series_partial: logi FALSE
#>   .. ..$ UnitTesting   : chr "TRUE"
#>   .. ..$ info          : chr "SampleDataMatrix"
#>   .. ..$ n             : chr "180"
#>   ..$ sampledata:An 'xts' object on 2007-06-20 22:00:00/2007-06-29 22:00:00 containing:
#>   Data: num [1:10, 1] 47.7 47.6 47.2 47.3 47.4 ...
#>  - attr(*, "dimnames")=List of 2
#>   ..$ : NULL
#>   ..$ : chr "High"
#>   Indexed by objects of class: [POSIXct,POSIXt] TZ: GMT
#>   xts Attributes:  
#> List of 6
#>   .. ..$ statement_id  : int 0
#>   .. ..$ series_names  : chr "sampledata"
#>   .. ..$ series_partial: logi FALSE
#>   .. ..$ UnitTesting   : chr "TRUE"
#>   .. ..$ info          : chr "SampleDataMatrix"
#>   .. ..$ n             : chr "180"
```

#### Simplify InfluxDB response

In case the InfluxDB response is expected to be a single series only, we can flatten the list (`simplifyList = TRUE`) to directly get to the data. This enhances a pipeable work flow.

``` r
result <- influx_select(con = con, 
                        db = "mydb", 
                        field_keys = "Open", 
                        measurement = "sampledata",
                        group_by =  "*",
                        limit = 10, 
                        order_desc = TRUE, 
                        return_xts = TRUE, 
                        simplifyList = TRUE)

str(result)
#> An 'xts' object on 2007-06-20 22:00:00/2007-06-29 22:00:00 containing:
#>   Data: num [1:10, 1] 47.7 47.6 47.2 47.2 47.2 ...
#>  - attr(*, "dimnames")=List of 2
#>   ..$ : NULL
#>   ..$ : chr "Open"
#>   Indexed by objects of class: [POSIXct,POSIXt] TZ: GMT
#>   xts Attributes:  
#> List of 6
#>  $ statement_id  : int 0
#>  $ series_names  : chr "sampledata"
#>  $ series_partial: logi FALSE
#>  $ UnitTesting   : chr "TRUE"
#>  $ info          : chr "SampleDataMatrix"
#>  $ n             : chr "180"
```

Contributions
-------------

This Git repository uses the [Git Flow](http://nvie.com/posts/a-successful-git-branching-model/) branching model (the [`git flow`](https://github.com/petervanderdoes/gitflow-avh) extension is useful for this). The [`dev`](https://github.com/dleutnant/influxdbr/tree/dev) branch contains the latest contributions and other code that will appear in the next release, and the [`master`](https://github.com/dleutnant/influxdbr) branch contains the code of the latest release, which is exactly what is currently on [CRAN](https://cran.r-project.org/package=influxdbr).

Contributing to this package is easy. Just send a [pull request](https://help.github.com/articles/using-pull-requests/). When you send your PR, make sure `dev` is the destination branch on the [influxdbr repository](https://github.com/dleutnant/influxdbr). Your PR should pass `R CMD check --as-cran`, which will also be checked by <a href="https://travis-ci.org/dleutnant/influxdbr">Travis CI</a> when the PR is submitted.

Code of condcut
---------------

Please note that this project is released with a [Contributor Code of Conduct](CONDUCT.md). By participating in this project you agree to abide by its terms.
