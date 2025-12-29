DuckDB R Tips

Erik Squires

Here’s a few quality of life tips I’ve gathered from working with R, RStudio and duckdb. I hope you find them useful.

Introduction

DuckDB is a very powerful in process database engine and data utility. It’s speed, full db features, ease of connection and ability to chew through very large data sets makes it an ideal companion to data scientists. I won’t go over all of it’s uses here, but in working with it to create a custom OLAP database at Uber for some rather involved forecast processes I came up with a list of tips I am sure can help many so I’m posting them all here. My work involved querying their data lake through a custom SQL wrapper, exploratory research and forecasting. Having a laptop DuckDB instance with 4 years of Uber rides, CPU and network telemetry was the single most important accelerator to my work there. After that what helped me a great deal was managing DuckDB connections.

If you write processes of any sort of complexity, with multiple scripts and/or multiple function libraries which access DuckDB data I think you’ll find my observations and tips useful. This will also help you make your R code easier to share with your colleagues.

Observations

DuckDB connections are nearly instant

DuckDB does not support multiple read/write connections

Together these offer a problem and an opportunity.

DB Setup

I found myself using my personal DB in a variety of different projects. Also, I wrote ETL scripts that would incrementally update a number of tables in them. For this I found it easier to keep the database itself in ~/db/ but use a soft link in each R project. Make sure to add *.db to .gitignore.

$ cd /dev/my_r_project
$ ln -s ~/db .

Helpful R Libraries

There are a few libraries you should get comfortable with.

config::get() reads config.yml from the project root, picking the default configuration unless otherwise told to. Here’s an example:

default:
  duckdb:
    path: "db"
    name: "dev.duckdb"

production:
  duckdb:
    path: "db"
    name: "prod.duckdb"

The idea of a YAML config file has been so useful it has been ported to most languages. It provides a hierarchical variable retrieval. Having a global cfg variable (a list in R terms) is very helpful. Assuming the config.yml above we can use:

# Use directly to avoid namespace conflicts
# library(config)
# library(here)

cfg <- config::get()

db_file <- here::here(cfg$duckdb$path, cfg$duckdb$name)

here() returns the root of your R project, no matter what file you are in, and works in R, Rmd and Qmd files reliably. This alone solves a lot of problems in maintaining your docs/ files consistent and source() libraries. Parameters are concatenated with the local OS path separator as needed (‘/’ or ‘\’).

Shiny

When you create variables in app.R you are already inside a local namespace, so if you want to read config.yml in one place and have your entire app know about it you need to explicitly put it in the global environment:

.GlobalEnv$cfg <- config::get()

Concurrency

The one major annoyance of DuckDB is that it only allows a single read/write connection, however it allows any number of simultaneous read only connections. This is not a problemm for simple scripts that open one connection at the top, reuse it throughout and then close before quitting. Unfortunately this leads us to a problem with attempting to write modular, reusable function libraries. We are left having to create some mechanism to test whether the connection has been opened or not in every single use. I propose that we take advantage of the fast open/close mechanics instead and get into the habit of using read-only connections by default in granular ways. For instance:

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

With the above two functions defined before accessing duck, you can now have multiple simultaneous connections :

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

Minimum R Script

library(DBI)
library(duckdb) # <-- Important for RStudio connection panel to work correctly
library(tidyverse)
# Optional either of:
# library(duckplyr) <-- More duckdb features but may not work everywhere dbplyr does.
# library(dbplyr)   <-- Reliable but not duckdb aware

# Loads our duck_connect_xx() functions
source(here::here("lib", "db_funcs.R"))

If you are not already familiar with dbplyr/duckplyr you should be. They push a lot of the work of dplyr down to the database level which can really accelerate your work and keep your memory footprint down.

Maybe don’t use R

If you find yourself doing ETL, then a number of data aggregations / migrations in R (or Python for that matter) you may be much better off managing your data herding with dbt. Think of it as Terraform for your data.

Maybe don’t ETL

A feature to lean into is that dbt and duckdb have extentions to read external databases (postgres, mysql, etc), files (parquet, csv, etc.) and remote sources via httpfs. These may eliminate the initial need to do an external ETL and keep your work 100% inside of duckdb and by extension in dbt.

Postgres has similar capabilities so look into the Foreign Data Wrapper.

Regardless of your data lake or desktop DB these options are all worth considering before reinventing the wheel with custom, brittle ETL scripts.

Data Transformation

dbt is mainly for working inside a single database/warehouse. However, once you have your initial data it does a fantastic job of managing table updates and views and will make it easy to move your db out of duck and into a shared corporate datastore.

dbt and your database are the right places to do data aggregation and data flows. If you find yourself doing sums, averages, percentiles, and various joins with categorical tables, or you’ve already found yourself making views and stored procedures to make your R code simpler and more consistent you are 100% in the land of dbt.

Where you want to keep R is in your statistical analysis, plotting, anything more complicated than calculating standard deviations. lm(), prophet, SARIMAX, estimating a t-distribution all belong in R.

Postgres or DuckDB?

There are many ways in which these two db’s can overlap. We can argue performance, but for me the real issue is the configuration and set up time. If you already have a PostgresDB and want to keep using it, please go ahead. Where duckdb shines is in going from zero to OLAP DB in a single line of R code. Users? Privileges? Haha! We don’t need no stinking user accounts. Tablespaces? Hahaha, that’s what we call the db file. Growth management… why? You need backups? Google drive!!

<img src="images/pirate_flag.png" style="width:50.0%"
alt="Pirate Flag" />

In all these ways DuckDB is the superior, single laptop analytical choice. The one and only one area I know of where Postgres, or Maria or any other RDBMS really can claim superiority is in managing multiple read write connections.

It is a perfectly reasonable approach to nurture your data science process With R and duckdb first, maybe adding dbt later as your scripts and data flows mature before pushing the ecosystem into a shared database.
