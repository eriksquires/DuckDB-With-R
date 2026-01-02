# DuckDB R Tips
Erik Squires

# Introduction

This guide is tailored to those of you who are
[DuckDB](https://duckdb.org/) users or DuckDB-curious and who are facing
challenges as your projects grow in complexity. If you find yourself
writing multiple scripts and/or multiple function libraries which access
DuckDB data I think you’ll find my observations and tips useful.

# DuckDB or Other?

DuckDB is a very powerful in process database engine and data utility.
It’s speed, full db features, ease of connection and ability to chew
through very large data sets makes it an ideal companion to data
scientists. Our data complexity often progresses from using CSV files
then RDS files and finally OMG, this is a lot of data in a lot of
tables!

My own experience was similar. While using DuckDB to create a custom
OLAP database at Uber for some rather involved forecast processes I came
up with a list of tips I am sure can help many so I’m posting them all
here. My work involved querying their data lake through a custom SQL
wrapper, exploratory research and forecasting. While Uber has an amazing
data lake infrastructure it was not ready for my use cases. Billions of
rows of data inserted per hour per table in addition to heavy internal
usage made even basic queries painfully slow. I solved my problems by
using DuckDB to create a custom OLAP DB. With 4 years of Uber rides, CPU
and network telemetry DuckDB was the single most important accelerator
to my work there.

DuckDB can often be a full-feature replacement for what we would do in a
corporate managed RDBMS. We can argue performance, but for me the real
benefits are the configuration and set up time. If you already have a
Postgres DB (or any other externally managed SQL datastore) and want to
keep using it, please go ahead. Where DuckDB shines is in going from
zero to OLAP DB in a single line of R code. Users? Privileges? Haha! We
don’t need no stinking user accounts. Tablespaces? Hahaha, that’s what
we call the db file. Growth management… why? You need backups? Google
Drive!!

<img src="images/pirate_flag.png" style="width:50.0%"
alt="Pirate Flag" />

In all these ways DuckDB is the superior, single laptop choice. The one
and only one area I know of where Postgres, or Maria or any other RDBMS
really can claim superiority is in supporting multiple read write
connections. As of this writing, there is
[MotherDuck](https://motherduck.com) for collaboration but it still
doesn’t support more than one writer at a time.

# Observations

- DuckDB connections are nearly instant
- Scans are very fast
- DuckDB does not support multiple read/write connections

Together these offer problems and opportunities.

# DB Setup

I chose to use a single DuckDB instance in a several different but
related (by data) projects. At first I found myself copying or moving
the db around to each new project but this was not manageable, and I
repeatedly lost track as to which copy was the most current. I ended up
with the following separation of concerns:

- The DB itself lived outside of R projects, in ~/db

- One R project was strictly for incremental ETL. One script per table,
  and a master ETL script I could quickly comment tables in/out.

- Many R projects shared the same db.

You may ask, why not just keep all my forecasts in one R project. They
were pretty wildly different and complicated, as well as would be handed
off to different teams for consumption. Attempting to put all R, and
reports in one project would not have been something I could manage
well, or explain to anyone. It would have been like storing all your
files in your root directory.

Though I only had a single db copy I used a soft link in each R project
folder, which I demonstrate below. Make sure to add **\*\*/db/** to
.gitignore so your git commits don’t accidentally commit the whole db.

``` bash
$ cd /dev/my_r_project
$ ln -s ~/db .
```

At about 10 GB, the DB was manageable, and transferable but definitely
too large for git.

## Helpful R Libraries

There are a few libraries you should get comfortable with.

### Config

**config::get()** reads config.yml from the project root, picking the
default configuration unless otherwise told to. Here’s an example:

``` yaml
default:
  DuckDB:
    path: "db"
    name: "dev.duckdb"

production:
  DuckDB:
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

db_file <- here::here(cfg$DuckDB$path, cfg$DuckDB$name)
```

I also like to use **here()** which returns the root of your R project,
no matter what file you are in, and works in R, Rmd and Qmd files
reliably. This alone solves a lot of problems in maintaining your docs/
files consistent and source() libraries. Parameters are concatenated
with the local OS path separator as needed (‘/’ or ‘\\’).

My advice is to keep a global cfg variable. It eliminates cluttering
your function signatures, or reloading in each function but also allows
you to programmatically modify those variables in your main script. The
same is true of having a global logger (see log4r) object.

### optparse

Options Parse, or `optparse` is a package which helps you read command
line arguments. It works well with config.

# The Concurrency Issue

The one major annoyance of DuckDB is that it only allows a single
read/write connection, however it allows any number of simultaneous read
only connections. This is not a problem for one-page scripts that open
one connection at the top, reuse it throughout and then close before
quitting. Unfortunately this doesn’t hold up when we are managing
complex work and need to write modular, reusable function libraries. Do
you pass the connection around? If so who and what opens it with the
correct write privileges? How many times must you check it to be sure?
Are you checking it the right way?

This also becomes a problem when you are editing your schema, or viewing
it in another tool like VS Code or the duckdb CLI and want to test your
R scripts a line at a time. You end up loading the CLI, making your
edit, exiting, then running your R code. Same thing happens if you are
editing a Quarto/Rmd document, but still debugging your underlying
libraries. The concurrency model can really get in your way without some
careful management.

To minimize both of these friction points at once I propose that we take
advantage of the fast open/close mechanics instead and get into the
habit of using read-only connections by default in granular ways. For
instance:

``` r
# Define read only and read/write functions for connection management
# Important that they test the db file location.  With DuckDB, if its' not there
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
  db_name <- here::here(cfg$DuckDB$path, cfg$DuckDB$name)
  # Return null if db_name does not exist
  
  if (!file.exists(db_name)) {
    warning(glue::glue("DuckDB database does not exist at {db_name}"))
    return(NULL)
  }
  con <- dbConnect(duckdb(), dbdir= db_name, read_only = TRUE)
  return(con)
}

duck_connect_rw <- function() {
  cfg <- config::get()
  db_name <- here::here(cfg$DuckDB$path, cfg$DuckDB$name)
  # Return null if db_name does not exist
  
  if (!file.exists(db_name)) {
    warning(glue::glue("DuckDB database does not exist at {db_name}"))
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

To be clear, you can’t write data with a read only connection. The
reason I propose this pattern is that it allows you to write modular
functions that can be reused in multiple scripts without worrying about
whether the connection is already open or not. Each function opens and
closes its own connection as needed. Since DuckDB connections are so
fast this is not a performance issue.

When you do need to write data, use duck_connect_rw() to get a
read/write connection. Just make sure that no other rw connections are
open at the same time. Usually we’ll find that we only need to write
data towards the end of a process after all the analysis is done. Even
so, do it in a tight code block like this:

``` r
df <- run_forecasts(...)

rw_con <- duck_connect_rw()
dbWriteTable(rw_con, name="results", df, append=TRUE)
dbDisconnect(rw_con)

# You can also write a custom append function: 

duck_append <- function(name, df) {
  rw_con <- duck_connect_rw()
  on.exit(dbDisconnect(rw_con), add=TRUE) # Forces connection closed on exit of function
  if (!dbExistsTable(rw_con, name)) {
    stop(glue::glue("{name} table does not exist, cannot append."))
  }
  dbWriteTable(rw_con, name= name, df, append=TRUE)
}
```

Either approach works, the point is that you prevent a RW connection
from malingering, forcing you to close it or restart your R session just
to get rid of it. I do this especially when writing Rmd/Qmd files,
because I’m usually debugging the data, db or underlying libraries at
the same time that I’m writing documentation. This makes debugging while
authoring much easier.

## Performance

### Iterations

Like most DBs, DuckDB maintains a cache of the DB data in the buffer
pool, so if you open and close a connection each time you are using the
db you lose that cache. However, DuckDB scans are fast, and your OS will
cache your db file so you often won’t feel this as a performance
penalty, until you do.

If you find yourself iterating over a large table you may want to keep
the same connection open throughout the iteration process. This advice
is not contradictory with the lessons above. If your iterations use
functions which also open your duck db they will get new connections to
the same cached DuckDB instance. The difference is now you can also use
your functions in test or other scripts without refactoring.

### Views

While duckdb caches file pages it does not cache view results. Each time
you access a view you’ll suffer the same compute time. For this reason,
maintaining a connection open during multiple view reads won’t help.
What will help however is to materialize the view results into a table.
Here’s an example in R/SQL:

``` r
# Creates a table that will persist across sessions and connections: 
dbExecute(con_rw, "CREATE OR REPLACE TABLE cached_view as SELECT * FROM my_view 
          WHERE state = 'TX'")

# Creates a temporary table, but only visible to this connection instance.
dbExecute(con_rw, "CREATE TEMP TABLE my_temp as SELECT * FROM my_view 
          WHERE state = 'TX'")
```

A **temporary table** is a real disk-backed table in the DB which exists
only in the scope of your connection. No other connection can see it,
and when the connection closes it vanishes into thin air. Temp tables
are incredibly useful constructs as they let us create large working
data sets entirely in the database, before bringing subsets into R for
analysis. This can save you time recomputing the view as well as let you
keep very large result sets in the DB until you are ready for them.

We’ll discuss how to leverage views even more in the section on dbplyr,
below.

# dbplyr

**dbplyr** pushes **dplyr** semantics such as `filter()`, `group_by()`,
and `summarize()` down to the DB, converting them to SQL for you.
`dbplyr` is generic and works with any database with an R::DBI driver.
Results are “lazy.” That is, you can express your data set as if you
were getting a data frame but until you call `collect()`, the data never
leaves the duck.

Half of the performance value of using duckdb comes from using dbplyr.
Explanations from here on are best done in code comments, below.

``` r
library(tidyverse)
library(dbplyr) # or dbplyr

# Assume we've loaded your custom DuckDB connection library here.


# Need RW connection because we will be writing a temp table. 
con_rw <- duck_connect_rw()

# Lets get US zip codes and whether they are on the coast line or not.
zip_codes <- tbl(con_rw, "zip_codes_us") %>%
  select(code, is_coastal)

# zip_codes is not a data frame, yet! 
# zip_codes is a tbl_dbi object. 
# Another way to think about it is that zip_codes is a SQL query that has not been 
# executed yet. 

df_temp <- tbl(con_rw, "my_view") %>%
  filter(state='TX') %>%
  left_join(zip_codes, by="code")

# df_temp is also a tbl_dbi object.  Notice we are joining to another
# tbl_dbi object. df_tmp is a new SQL query that's yet to be executed. 

# Prints the SQL would be executed
show_query(df_temp)  

# Materialize the data into a data frame, finally!
coastal_df <- df_temp %>%
  filter(is_coastal == TRUE) %>%
  collect() # <-- Without this, coastal_df would be yet another tbl_dbi object.

# We can reuse the temp table via df_temp as much as we need to. 


dbDisconnect(con_rw)

# Access to all tbl_dbi objects will now fail. 

# Materialized data frames like coastal_df however are just data frames and will
# remain.
```

You may ask “can’t I do this all with R?” and of course you can, but in
this example we leveraged DuckDB for the filtering and join while
staying within dplyr semantics.

## dbplyr and Temp Tables

As we mentioned above, using views which have long compute times may be
painful and leave you wanting to use a temporary table. Let’s convert
`df_temp`, above, to be backed by a temp table by adding `collect()`:

``` r
df_temp <- tbl(con_rw, "my_view") %>%
  filter(state='TX') %>%
  left_join(zip_codes, by="code") %>%
  collect()
```

When you call `compute()` you materialize the data set but to a temp
table instead of a data frame. The point is, again, to let DuckDB handle
large dataset manipulation and storage.

**Takeaway:** Converting a result to a temp table is one line of code.
Don’t over think it. You can always go back and use temp tables when you
feel the pain.

## duckplyr

Duckplyr is meant to function as dbplyr for external **files** as
opposed to a **Database.** Think especially parquet files but also CSV
and JSON. A key difference is the object you work with.

dbplyr::tbl() returns a tbl_dbi object. Essentially a SQL statement that
has yet to be executed.

If your data lives in files and need large dataset handling this is your
toolset. Think of a duckdb_tibble as a tibble that is backed by DuckDB
instead of a pure R tibble or data frame.

Example:

``` r
library(duckplyr)

# From a db we would use tbl() but from individual files 
# we use the read_*_duckdb() functions to start.
employees <- read_parquet_duckdb("employees.parquet")

salary <- read_parquet_duckdb("salaries.parquet")

payroll <- employees %>% 
  left_join(salary, on = 'employee_id')

# Use employees, salary and payroll like any other data frame. 
```

In the case above you use DuckDB like a super file scanner. Payroll is
joined and returned by DuckDB, not R. Another potential benefit is using
duckdb to glue together multiple data files with one line:

``` r
my_dat <- read_parquet_duckdb("data/*.parquet")
```

So long as your file schema is homogenous you’ll get a single, db backed
object without having to ingest it all yet.

Obviously, duckdb is overkill for a CSV file with 4 columns and a couple
of thousand rows in a couple of files. It’s when your dataset is large,
files are many and want to keep dplyr semantics that duckplyr shines.

# RStudio Connections Panel Hiccups

It IS possible to get a valid connection but not have the RStudio
Connections panel work correctly. The reason is that schema
introspection methods are tied to the back-end driver package, but are
not needed for basic DBI functions like dbConnect(), dbGetQuery() or
dbExecute(). As a result, without `duckdb` (or `RPostgres`) explicitly
loaded the Connections panel won’t fully populate. The bad part about
this is there’s no warning message anywhere to tell you that the
Connections panel is only half working. This is so frustrating because
as far as you an tell, your database is working, the tables are there
and you can use your data exactly as expected.

``` r
library(DBI)

# We use duckdb, but never load duckdb. 
con <- dbConnect(duckdb::duckdb(), ...)

# The Connections panel will display something, but not everything. 
# Especially if you use schemas besides the default.

# This will work just fine, without duckdb loaded, relies on DBI alone.
df <- dbGetQuery(con, ....)
```

Here’s what you could do instead:

``` r
library(DBI)
library(duckdb) # Or whatever your actual DB backend is, like RPostgres

# Same exact connection string, but loading the duckdb (above) ensures the 
# RStudio Connections panel will work as expected. 
con <- dbConnect(duckdb::duckdb(), ...)

# Now the Connections panel should show you everything
# in your Duck DB

# This will work just fine. 
df <- dbGetQuery(con, ....)
```

# Data Migration Tips

Kind of related to using duckdb with R is the overall process of data
transformation. If you are using duckdb (or any db really) with R you
are already involved in extraction and loading or extraction,
transformation and loading (ETL). Somehow you have to go from data which
comes from a primary source to data which your analysis and analytical
models can take advantage of. In addition to the work you care about you
may also be responsible for moving the data along to others to use with
BI tools instead of R or Python.

The modern data scientist/analytics engineer has to fend for themselves
more often than not. It is rare we have the luxury of a data team to
just hand us tables on a silver platter. If that is you and you have
started going down the road of self sufficiency I have some pointers.

## Maybe don’t ETL

One of the first things we need to do is get raw, dirty data. This is
often done with custom ETL scripts (in python, R, Perl, bash, Ruby,
etc.) which read from external databases, files, APIs and other sources
and put them somwhere R can get to them. While this is a very common
approach it does have some downsides. Custom ETL scripts are often
brittle, require maintenance and monitoring, and can be difficult to
share with colleagues.

Before writing and maintaining ETL scripts you should consider that most
modern RDBMS systems usually include extensions to read external
databases directly. DuckDB for instance can connect to remote DBs
(postgres, mysql, etc), files (parquet, csv, etc.) and remote sources
via httpfs. These may eliminate the initial need to do an external ETL
and keep your work 100% inside of DuckDB and by extension in dbt.
Postgres, as an example, has a Foreign Data Wrapper which serves this
purpose nicely.

Regardless of your data lake or desktop DB these options are all worth
considering before reinventing the wheel with custom, brittle ETL
scripts. I cannot stress this enough: **Data Connections are much more
reliable and less maintenance than external, script driven (python,
bash, Perl, R) ETL jobs.**

Truthfully you won’t always be able to avoid maintaining your own ETL
jobs. Sometimes that’s the fastest route to getting results, but it’s
worth stopping to ask your colleagues what options exist before you dive
in.

## Maybe don’t use R for Data Transformation

A database is the right places to do data aggregation and data
migration. If you find yourself doing sums, averages, percentiles, and
various joins with categorical tables, or you’ve found yourself making R
function libraries to return consistent data sets you are 100% in the
land of databases and dbt (data build tool). Even if you are not ready
for dbt yet, pushing transformations into views instead of R code is a
big improvement in terms of performance and maintainability.

While we’ve shown how to use dbplyr for data transformation but you
should also consider whether you need to bring in a tool like DBT. DBT
is like Terraform but for analytic databases and workflows.

The big feature of dbt is that it is aware of dependencies between
tables and views and will build them in the correct order. This is a big
help when you have many interdependent views and tables. This is no
longer something you have to bake into your R code. You can also mix R
and dbt as needed. Consider a situation when you have to run R in the
middle. Let’s imagine you are running a forecast, which depends on
several upstream tables but there are also a number of financial table
that must be updated after the forecast to feed the Business
Intelligence (BI) charts. For this example, table C relies on tables A
and B. Table D is built by R. The BI tables (E, F, etc) depend on D. You
can manage this easily with dbt and R like so:

``` bash
$ dbt --select +C        # Builds A, B and C
$ Rscript run_forecast.R # Depends on A, B and C. Writes D
$ dbt --select D+        # Builds E, F, etc.
```

This is a very clean way to manage interleaved R and SQL work. Of
course, you also sidestep any concurrency issues. Set dbt threads = 1 in
profiles.yml to avoid any possible read/write conflicts, and let dbt
manage the ordering of work in the db. I think you will find that
building dbt models reduced the amount of work you must do in R (or
Python, etc.) with the same reliability and flexibility. When the data
governance people come to knock on your door you’ll be able to produce a
nice lineage graph showing how data flows through your system, as well
as documentation in schema.yml.

## DBT and Materialized Views

We cover views and temp tables above. If you have views which:

- Don’t have to be up to the minute
- Take a while to compute
- Are used by many, like BI dashboards

You may want to go ahead and let DBT create them for you overnight.

**Summary**: Where you want to keep R is in your statistical analysis,
plotting, anything more complicated than calculating standard
deviations. lm(), prophet, SARIMAX, estimating a t-distribution all
belong in R. It is a perfectly reasonable approach to nurture your data
science process with R and DuckDB first, maybe adding dbt later as your
scripts and data flows mature before pushing the ecosystem into a shared
database.

## OpenDBT

As I was interacting online I realized there’s [a fork of dbt called
OpenDBT](https://github.com/memiiso/opendbt). This is driven partly by
the fear of a merger but also by a desire to enhance the core
functionality. Among the features being actively worked on is built-in
extraction and load.

# Conclusion

There’s no one right way to manage DuckDB or data migration or code
complexity. I’ve shown you a number of options which I hope will be
useful to you as your project grows in complexity but none are required.
You don’t have to use a database. You don’t have to use `dbplyr` or
temporary tables but having those tools in your bag should be useful. My
own view is that what we call “quality of life” improvements are really
core productivity requirements so it is worth recognizing your own pain
points and solving them when you feel them. When you do you’ll find
yourself consistently surprising yourself with your own level of
productivity and satisfaction with the work you are doing.
