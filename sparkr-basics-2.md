# SparkR Basics II: Essential DataFrame Operations
Sarah Armstrong, Urban Institute  
June 28, 2016  



**Last Updated**: August 17, 2016


**Objective**: The SparkR DataFrame (DF) API supports a number of operations to do structured data processing. These operations range from the simple tasks that we used in the [SparkR Basics I](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/sparkr-basics-1.md) tutorial (e.g. counting the number of rows in a DF using `nrow`) to more complex tasks like computing aggregate data. This tutorial discusses the key DF operations for processing tabular data in the SparkR environment, the different types of DF operations and how to perform these operations efficiently. In particular, this tutorial discusses:

* Computing aggregations for a specified list of columns across an entire DF
* Computing aggregations for a specified list of columns across entries of a DF that share a common identifier
* Arranging (ordering) rows in a DF
* Appending a column to a DF
* User-defined functions (UDFs)
* Types of DF operations
* DataFrame persistence: what is persistence and when should we persist DFs?
* Converting a SparkR DF to a local R data.frame

**SparkR/R Operations Discussed**: `agg`, `summarize`, `showDF`, `avg`, `mean`, `sd`, `stddev`, `stddev_samp`, `stddev_pop`, `var`, `variance`, `var_samp`, `var_pop`, `countDistinct`, `n_distinct`, `first`, `last`, `max`, `min`, `sum`, `arrange`, `orderBy`, `withColumn`, `withColumnRenamed`, `persist`, `cache`, `unpersist`, `take`, `collect`

***

:heavy_exclamation_mark: **Warning**: Before beginning this tutorial, please visit the SparkR Tutorials README file (found [here](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/README.md)) in order to load the SparkR library and subsequently initiate a SparkR session.



The following error indicates that you have not initiated a SparkR session:


```r
Error in getSparkSession() : SparkSession not initialized
```

If you receive this message, return to the SparkR tutorials [README](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/README.md) for guidance.

***

**Read in initial data as DF**: Throughout this tutorial, we will use the loan performance example dataset that we exported at the conclusion of the [SparkR Basics I](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/sparkr-basics-1.md) tutorial.


```r
df <- read.df("s3://sparkr-tutorials/hfpc_ex", header = "false", inferSchema = "true")
```



_Note_: documentation for the quarterly loan performance data can be found at http://www.fanniemae.com/portal/funding-the-market/data/loan-performance-data.html.

***


### Aggregating

Computing aggregations across a dataset is a basic goal when working with tabular data and, because our data is distributed across nodes, we must explicitly direct SparkR to perform an aggregation if we want to compute and return a summary statistic. Both the `agg` and `summarize` operations achieve this by computing aggregations of DF entries based on a specified list of columns. For example, we can return the mean loan age for all rows in the DF `df` with:


```r
df1 <- agg(df, loan_age_avg = avg(df$loan_age))
showDF(df1) # Prints the first numRows rows of a DF; the default is numRows = 20
## +------------------+
## |      loan_age_avg|
## +------------------+
## |29.491677307393264|
## +------------------+
```

We can compute a number of aggregations by specifying them in `agg` or `summarize`. The following list illustrates the types of summary statistics that can be computed and is not exhaustive:

* `avg`, `mean`: return the mean of a DF column
* `sd`, `stddev`, `stddev_samp`: return the unbiased sample standard deviation in the values of a DF column
* `stddev_pop`: returns the population standard deviation in a DF column
* `var`, `variance`, `var_samp`: return the unbiased variance of the values in a DF column
* `var_pop`: returns the population variance of the values in a DF column
* `countDistinct`, `n_distinct`: return the number of distinct items in a DF column
* `first`, `last`: return the first and last item in a DF column, respectively
* `max`, `min`: return the maximum and minimum of the values in a DF column
* `sum`: returns the sum of all values in a DF column

***


### Grouping

If we want to compute aggregations across the elements of a dataset that share a common identifier, we can achieve this embedding the `groupBy` operation in `agg` or `summarize`. For example, the following `agg` operation returns the mean loan age and the number of observations for each distinct `"servicer_name"` in the DataFrame `df`:


```r
gb_sn <- groupBy(df, df$servicer_name)
df2 <- agg(gb_sn, loan_age_avg = avg(df$loan_age), count = n(df$loan_age))
head(df2)
##                                servicer_name loan_age_avg count
## 1                                   EVERBANK    144.31667   180
## 2                         QUICKEN LOANS INC.    160.00000     4
## 3       FLAGSTAR CAPITAL MARKETS CORPORATION     99.69091    55
## 4                   NATIONSTAR MORTGAGE, LLC    163.95785   261
## 5 FIRST TENNESSEE BANK, NATIONAL ASSOCIATION     23.45820 12774
## 6                      BANK OF AMERICA, N.A.     20.97741 34704
```



***


### Arranging (Ordering) rows in a DataFrame

The operations `arrange` and `orderBy` allow us to sort a DF by a specified list of columns. If we want to sort the DataFrame that we just specified, `df2`, we can arrange the rows of `df2` by `"loan_age_avg"` or `"count"`. Note that the default for `arrange` is to order the row values as ascending:


```r
df2_a1<- arrange(df2, desc(df2$loan_age_avg))  # List servicers by descending mean loan age values
head(df2_a1)
##                             servicer_name loan_age_avg count
## 1                  FREEDOM MORTGAGE CORP.     184.5000     2
## 2                    DITECH FINANCIAL LLC     184.2703  1106
## 3   MATRIX FINANCIAL SERVICES CORPORATION     168.0455    22
## 4 FANNIE MAE/SETERUS, INC. AS SUBSERVICER     165.9603   504
## 5                NATIONSTAR MORTGAGE, LLC     163.9579   261
## 6               GREEN TREE SERVICING, LLC     162.4933  1409

df2_a2 <- arrange(df2, df2$count) # List servicers by ascending count values
head(df2_a2)
##                           servicer_name loan_age_avg count
## 1             OCWEN LOAN SERVICING, LLC    160.50000     2
## 2                FREEDOM MORTGAGE CORP.    184.50000     2
## 3                    QUICKEN LOANS INC.    160.00000     4
## 4           IRWIN MORTGAGE, CORPORATION     38.84615    13
## 5 MATRIX FINANCIAL SERVICES CORPORATION    168.04545    22
## 6  FLAGSTAR CAPITAL MARKETS CORPORATION     99.69091    55
```

We can also specify ordering as logical statements. The following expressions are equivalent to those in the preceding example:


```r
df2_a3 <- arrange(df2, "loan_age_avg", decreasing = TRUE)
head(df2_a3)
##                             servicer_name loan_age_avg count
## 1                  FREEDOM MORTGAGE CORP.     184.5000     2
## 2                    DITECH FINANCIAL LLC     184.2703  1106
## 3   MATRIX FINANCIAL SERVICES CORPORATION     168.0455    22
## 4 FANNIE MAE/SETERUS, INC. AS SUBSERVICER     165.9603   504
## 5                NATIONSTAR MORTGAGE, LLC     163.9579   261
## 6               GREEN TREE SERVICING, LLC     162.4933  1409

df2_a4 <- arrange(df2, "count", decreasing = FALSE)
head(df2_a4)
##                           servicer_name loan_age_avg count
## 1             OCWEN LOAN SERVICING, LLC    160.50000     2
## 2                FREEDOM MORTGAGE CORP.    184.50000     2
## 3                    QUICKEN LOANS INC.    160.00000     4
## 4           IRWIN MORTGAGE, CORPORATION     38.84615    13
## 5 MATRIX FINANCIAL SERVICES CORPORATION    168.04545    22
## 6  FLAGSTAR CAPITAL MARKETS CORPORATION     99.69091    55
```



***


### Append a column to a DataFrame

There are various reasons why we might want to introduce a new column to a DataFrame. A simple example is creating a new variable using our data. In the SparkR environment, this could be acheived by appending an existing DF using the `withColumn` operation.

For example, the values of the `"loan_age"` column in `df` are the number of calendar months since the first full month that the mortgage loan accrues interest. If we want to convert the unit of time for loan age from calendar months to years and work with this measure as a variable in our analysis, we can evaluate the following `withColumn` expression:


```r
df3 <- withColumn(df, "loan_age_yrs", df$loan_age * (1/12))
head(df3)
##        loan_id     period servicer_name new_int_rt act_endg_upb loan_age
## 1 404371459720 09/01/2005                     7.75     79331.20       67
## 2 404371459720 10/01/2005                     7.75     79039.52       68
## 3 404371459720 11/01/2005                     7.75     79358.51       69
## 4 404371459720 12/01/2005                     7.75     79358.51       70
## 5 404371459720 01/01/2006                     7.75     78365.73       71
## 6 404371459720 02/01/2006                     7.75     78365.73       72
##   mths_remng aj_mths_remng dt_matr cd_msa delq_sts flag_mod cd_zero_bal
## 1        293           286 02/2030      0        5        N          NA
## 2        292           283 02/2030      0        3        N          NA
## 3        291           287 02/2030      0        8        N          NA
## 4        290           287 02/2030      0        9        N          NA
## 5        289           277 02/2030      0        0        N          NA
## 6        288           277 02/2030      0        1        N          NA
##   dt_zero_bal loan_age_yrs
## 1                 5.583333
## 2                 5.666667
## 3                 5.750000
## 4                 5.833333
## 5                 5.916667
## 6                 6.000000
```

Note that `df3` contains every column originally included in `df`, as well as the column `"loan_age_yrs"`.

We can also rename a DF column using the `withColumnRenamed` operation as we discussed in the [SparkR Basics I](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/sparkr-basics-1.md) tutorial. The following expression returns a DF that is equivalent to `df`, except for the fact that we have renamed `"servicer_name"` to `"servicer"`.


```r
df4 <- withColumnRenamed(df, "servicer_name", "servicer")
head(df4)
##        loan_id     period servicer new_int_rt act_endg_upb loan_age
## 1 404371459720 09/01/2005                7.75     79331.20       67
## 2 404371459720 10/01/2005                7.75     79039.52       68
## 3 404371459720 11/01/2005                7.75     79358.51       69
## 4 404371459720 12/01/2005                7.75     79358.51       70
## 5 404371459720 01/01/2006                7.75     78365.73       71
## 6 404371459720 02/01/2006                7.75     78365.73       72
##   mths_remng aj_mths_remng dt_matr cd_msa delq_sts flag_mod cd_zero_bal
## 1        293           286 02/2030      0        5        N          NA
## 2        292           283 02/2030      0        3        N          NA
## 3        291           287 02/2030      0        8        N          NA
## 4        290           287 02/2030      0        9        N          NA
## 5        289           277 02/2030      0        0        N          NA
## 6        288           277 02/2030      0        1        N          NA
##   dt_zero_bal
## 1            
## 2            
## 3            
## 4            
## 5            
## 6
```

When using either `withColumn` or `withColumnRenamed`, we could simply replace our initial DF. For example, we could rename `"servicer_name"` by simply changing the name of the DF that we save to, i.e. `df <- withColumnRenamed(df, "servicer_name", "servicer")`. Note: do this _only_ if you do not need to retain your initial DF.


***


### Types of SparkR operations

Throughout this tutorial, as well as in the [SparkR Basics I](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/sparkr-basics-1.md) tutorial, you may have noticed that some operations result in a new DataFrame (e.g. `agg`) and some return an output (e.g. `head`). SparkR operations can be classified as either:

* __transformations__: those operations that return a new SparkR DataFrame; or,
* __actions__: those operations that return an output.

A fundamental characteristic of Apache Spark that allows us SparkR-users to perform efficient analysis on massive data is that transformations are lazily evaluated, meaning that SparkR delays evaluating these operations until we direct it to return some ouput (as communicated by an action operation). We can intuitively think of transformations as instructions that SparkR acts on only once its directed to return a result.


This lazy evaluation strategy (1) reduces the number of processes SparkR is required to complete and (2) allows SparkR to interpret an entire set of instructions before acting, and then make processing decisions that are obscured from SparkR-users in order to further optimize the evaluation of the expressions.

***


### DataFrame Persistence

Note that, in this tutorial, we have been saving transformations (e.g. `withColumn`) in the format `dfi` since, as we discussed in the preceding section, SparkR saves a transformation as a SparkR DataFrame, which is distinct from an R data.frame. We store the instructions communicated by a transformation as a DataFrame. An R data.frame, conversely, is an actual data structure defined by a list of vectors.

We saved the first transformation included in this tutorial, using `read.df` to read in our example data, as `df`. This operation itself does not load data into SparkR. Instead, `df` consists of instructions communicating to SparkR that the data should be read in and how SparkR should interpret the data as it is read in. Each time we direct SparkR to evaluate the expressions:


```r
head(df, num = 5)
head(df, num = 10)
```

SparkR would:

1. read in the data as a DataFrame,
2. look for the first five (5) rows of the DF,
3. return the first five (5) rows of the DF,
4. read in the data as a DF,
5. look for the first ten (10) rows of the DF and
6. return the first ten (10) rows of the DF

Note that nothing is stored since the `df` is not data! This would be incredibly inefficient if not for the `cache` operation, which directs each node in our cluster to store in memory any partitions of a DF that it computes (in the course of evaluating an action) and then to reuse this cache of the partitions in subsequent actions evaluated on that DF (or DFs derived from it).


By caching a given DataFrame, we can ensure that future actions on that DF (or those derived from it) are evaluated much more efficiently. Both `cache` and `persist` can be used to cache a DataFrame. The `cache` operation stores a DF in memory, while `persist` allows SparkR-users to persist a DataFrame using different storage levels (i.e. store to disk, memory or both). The default storage level for `persist` is memory only and, at this storage level, `persist` and `cache` are equivalent operations. More often than not, we can simply use `cache`: if our DataFrames can fit in memory only, then we should exclusively store DFs in memory since this is the most CPU-efficient storage option.


Now that we have some understanding of how DataFrame persistence works in SparkR, let's see how this operation affects the processes in the preceding example. By including `cache` with our expressions as


```r
df_ <- read.df("s3://sparkr-tutorials/hfpc_ex", header = "false", inferSchema = "true")
cache(df_)
head(df_, num = 5)
head(df_, num = 10)
```

The steps performed by SparkR change to:

1. read in the data as a DF,
2. look for the first five (5) rows of the DF,
3. return the first five (5) rows of the DF,
4. cache the DF
5. look for the first ten (10) rows of the DF (using the cache) and
6. return the first ten (10) rows of the DF.

While the number of steps required remains six (6), the time required to `cache` a DF once is significantly less than that required to read in data as a DF several times. If we continuited to perform actions on `df_`, clearly directing SparkR to cache the DF would reduce our overall evaluation time. We can direct SparkR to stop persisting a DataFrame with the `unpersist` operation:


```r
unpersist(df_)
```

Be sure to `unpersist` a DF if you are not continuing to reference it - minimizing the number of DFs stored in memory at a given time will help SparkR to perform more efficiently.


Let's compare the time elapsed in evaluating the following expressions with and without persistence:


```r
# Uncached
.df <- read.df("s3://sparkr-tutorials/hfpc_ex", header = "false", inferSchema = "true")
system.time(ncol(.df))
##    user  system elapsed 
##   0.020   0.000   0.021
system.time(nrow(.df))
##    user  system elapsed 
##   0.004   0.000   0.200
system.time(head(agg(groupBy(.df, .df$servicer_name), loan_age_avg = avg(.df$loan_age))))
##    user  system elapsed 
##   0.012   0.000   3.922
rm(.df)

# Cached
.df <- read.df("s3://sparkr-tutorials/hfpc_ex", header = "false", inferSchema = "true")
cache(.df)
## SparkDataFrame[loan_id:bigint, period:string, servicer_name:string, new_int_rt:double, act_endg_upb:double, loan_age:int, mths_remng:int, aj_mths_remng:int, dt_matr:string, cd_msa:int, delq_sts:string, flag_mod:string, cd_zero_bal:int, dt_zero_bal:string]
system.time(ncol(.df))
##    user  system elapsed 
##   0.024   0.000   0.026
system.time(nrow(.df))
##    user  system elapsed 
##   0.008   0.000   0.145
system.time(head(agg(groupBy(.df, .df$servicer_name), loan_age_avg = avg(.df$loan_age))))
##    user  system elapsed 
##   0.008   0.000   1.545
unpersist(.df)
## SparkDataFrame[loan_id:bigint, period:string, servicer_name:string, new_int_rt:double, act_endg_upb:double, loan_age:int, mths_remng:int, aj_mths_remng:int, dt_matr:string, cd_msa:int, delq_sts:string, flag_mod:string, cd_zero_bal:int, dt_zero_bal:string]
rm(.df)
```

The first thing you may notice is that the evaluation time for `ncol(.df)` is approximately the same with and without persistence. Remember from our the discussion above that SparkR caches a DF at the first action operation using the DF. So, the evaluation time for `ncol(.df)` with persistence is not noticeably smaller in value since it is the first action that uses `.df`. However, we can see that the evaluation time for each subsequent expression is significantly less when we cache `.df`, relative to when we do not. Our example dataset is actually quite small relative to the massive datasets that SparkR allows us to work with. Consider how essential persistence would be if we were peforming analysis on 15 years worth of quarterly loan performance data. Intelligently caching and unpersisting DFs would clearly make our analysis more efficient in that case.

***


### Converting a SparkR DataFrame to a local R data.frame

If we wanted to work the first five (5) rows of 'df' as a local R data.frame, we could use the `take` operation as follows:


```r
df_loc <- take(df, num = 5) # Creates a local data.frame `df_loc`
df_loc
##        loan_id     period servicer_name new_int_rt act_endg_upb loan_age
## 1 404371459720 09/01/2005                     7.75     79331.20       67
## 2 404371459720 10/01/2005                     7.75     79039.52       68
## 3 404371459720 11/01/2005                     7.75     79358.51       69
## 4 404371459720 12/01/2005                     7.75     79358.51       70
## 5 404371459720 01/01/2006                     7.75     78365.73       71
##   mths_remng aj_mths_remng dt_matr cd_msa delq_sts flag_mod cd_zero_bal
## 1        293           286 02/2030      0        5        N          NA
## 2        292           283 02/2030      0        3        N          NA
## 3        291           287 02/2030      0        8        N          NA
## 4        290           287 02/2030      0        9        N          NA
## 5        289           277 02/2030      0        0        N          NA
##   dt_zero_bal
## 1            
## 2            
## 3            
## 4            
## 5
```

Because `df_loc` is a normal R data.frame, we can work with it just as we would normally in RStudio, using R. The operation `collect` also creates an R data.frame. However, it coerces __all__ elements of a DF into a data.frame and, therefore, this should only be done if you can reasonably assume that the data called by the DF can fit onto a single node.


One way that we can safely use `collect` is to extract an aggregation as a value type in SparkR. Earlier in this tutorial, we computed the mean loan age across every entry of `df` with the transformation `df1 <- agg(df, loan_age_avg = avg(df$loan_age))`. This, however, returns a DF instead of a value (because it is a transformation!). Fortunately, we can use `collect` to extract the mean as a value data type as follows:


```r
loan_age_avg_ <- collect(df1) # Explicitly written expression

loan_age_avg <- loan_age_avg_[[1]]
loan_age_avg
## [1] 29.49168

typeof(loan_age_avg)
## [1] "double"
rm(loan_age_avg)

loan_age_avg <- collect(df1)[[1]] # Embedded expression
loan_age_avg
## [1] 29.49168
typeof(loan_age_avg)
## [1] "double"
```

Notice that we can direct SparkR to do this through either set of expressions above, with the process being written explicitly or implicitly. If we had not already defind `df1`, we could have directed SparkR to compute this value in a single line with the expression `loan_age_avg <- collect(agg(df, loan_age_avg = avg(df$loan_age)))[[1]]`.


__End of tutorial__ - Next up is [Subsetting SparkR DataFrames](https://github.com/UrbanInstitute/sparkr-tutorials/blob/master/subsetting.md)
