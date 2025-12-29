# DuckDB_R_Tips
Erik Squires

# DuckDB R Tips

Here’s a few quality of life tips I’ve gathered from working with R,
RStudio and duckdb. I hope you find them useful.

## Introduction

DuckDB is a very powerful in process database engine and data utility.
It’s speed, full db features, ease of connection and ability to chew
through very large data sets makes it an ideal companion to data
scientists. I won’t go over all of it’s uses here, but in working with
it to create a custom OLAP database at Uber for some rather involved
forecast processes I came up with a list of tips I am sure can help many
so I’m posting them all here. My work involved querying their data lake
through a custom SQL wrapper, exploratory research and forecasting.
Having a laptop DuckDB instance with 4 years of Uber rides, CPU and
network telemetry was the single most important accelerator to my work
there. After that what helped me a great deal was managing DuckDB
connections.

If you write processes of any sort of complexity, with multiple scripts
and/or multiple function libraries which access DuckDB data I think
you’ll find my observations and tips useful. This will also help you
make your R code easier to share with your colleagues.

## Observations

- DuckDB connections are nearly instant

- DuckDB does not support multiple read/write connections

Together these offer a problem and an opportunity.

## DB Setup

I found myself using my personal DB in a variety of different projects.
Also, I wrote ETL scripts that would incrementally update a number of
tables in them. For this I found it easier to keep the database itself
in ~/db/ but use a soft link in each R project. Make sure to add \*.db
to .gitignore.

``` bash
$ cd /dev/my_r_project
$ ln -s ~/db .
```

## Helpful R Libraries

There are a few libraries you should get comfortable with.

**config::get()** reads config.yml from the project root, picking the
default configuration unless otherwise told to. Here’s an example:

``` yaml
default:
  duckdb:
    path: "db"
    name: "dev.duckdb"

production:
  duckdb:
    path: "db"
    name: "prod.duckdb"
```

The idea of a YAML config file has been so useful it has been ported to
most languages. It provides a hierarchical variable retrieval. Having a
global cfg variable (a list in R terms) is very helpful. Assuming the
config.yml above we can use:

``` r
# Use directly to avoid namespace conflicts
# library(config)
# library(here)

cfg <- config::get()

db_file <- here::here(cfg$duckdb$path, cfg$duckdb$name)
```

here() returns the root of your R project, no matter what file you are
in, and works in R, Rmd and Qmd files reliably. This alone solves a lot
of problems in maintaining your docs/ files consistent and source()
libraries. Parameters are concatenated with the local OS path separator
as needed (‘/’ or ‘\\’).

### Shiny

When you create variables in app.R you are already inside a local
namespace, so if you want to read config.yml in one place and have your
entire app know about it you need to explicitly put it in the global
environment:

``` r
.GlobalEnv$cfg <- config::get()
```

## Concurrency

The one major annoyance of DuckDB is that it only allows a single
read/write connection, however it allows any number of simultaneous read
only connections. This is not a problemm for simple scripts that open
one connection at the top, reuse it throughout and then close before
quitting. Unfortunately this leads us to a problem with attempting to
write modular, reusable function libraries. We are left having to create
some mechanism to test whether the connection has been opened or not in
every single use. I propose that we take advantage of the fast
open/close mechanics instead and get into the habit of using read-only
connections by default in granular ways. For instance:

``` r
# Define read only and read/write functions for connection management
# Important that they test the db file location.  With Duckdb, if its' not there
# dbConnect() will create the file, leading you down a wild goose chase. 
# The file will exist, but your db will be empty, giving you a panic attack
# when you attempt to debug the problem. 
#
# I suggest putting these in lib/ and sourcing this file as needed. 
#

library(DBI)
library(duckdb)

duck_connect_ro <- function() {
  cfg <- config::get()
  db_name <- here::here(cfg$duckdb$path, cfg$duckdb$name)
  # Return null if db_name does not exist
  
  if (!file.exists(db_name)) {
    warn("DuckDB database does not exist at {db_name}")
    return(NULL)
  }
  con <- dbConnect(duckdb(), dbdir= db_name, read_only = TRUE)
  return(con)
}

duck_connect_rw <- function() {
  cfg <- config::get()
  db_name <- here::here(cfg$duckdb$path, cfg$duckdb$name)
  # Return null if db_name does not exist
  
  if (!file.exists(db_name)) {
    warn("DuckDB database does not exist at {db_name}")
    return(NULL)
  }
  con <- dbConnect(duckdb(), dbdir= db_name, read_only = FALSE)
  return(con)
}
```

With the above two functions defined before accessing duck, you can now
have multiple simultaneous connections :

``` r
func_a <- function (a, b, c) {
  con <- duck_connect_ro()
  
  # Use the connection
  # 
  # Close the connection before returning
  dbDisconnect(con)
  
  return(my_list)
}

func_b <- function (a, b, c) {
  con <- duck_connect_ro()
  
  # Use the connection
  #
  # Call func_a which also creates a connection
  list <- func_a(1, 2, 3)
  #
  # Close the connection before returning
  dbDisconnect(con)
  
  return(my_df)
}

df <- func_b(4,5,6)
```

# Minimum R Script

``` r
library(DBI)
library(duckdb) # <-- Important for RStudio connection panel to work correctly
library(tidyverse)
# Optional either of:
# library(duckplyr) <-- new but growing
# library(dbplyr) <-- solid but less full featured

# Loads our duck_connect_xx() functions
source(here::here("lib", "db_funcs.R"))
```

If you are not already familiar with dbplyr/duckplyr you should be. They
push a lot of the work of dplyr down to the database level which can
really accelerate your work and keep your memory footprint down.
