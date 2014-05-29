





# data.table
## build to reduce 2 types of time
1. programming time (easier to write, read, debug and maintain)
2. compute time

## data.table, just like a data.frame. However :
1. it never has rownames. Instead it may have **one key of one or more columns**. 
This key can be used for row indexing instead of rownames, and **duplicate key values are allowed**.
2. it has enhanced functionality in `[.data.table` for fast joins of keyed tables, 
fast aggregation, fast last observation carried forward (LOCF) and fast 
add/modify/delete of columns by reference with no copy at all.


 
## Basic syntax
x[i, j, by] - Take `x`, subset rows using `i`, then calculate `j` grouped by `by`
- i: *Integer, logical* or *character vector*, *expression of column names, list* or *data.table*
   - Integer and logical vectors work the same way they do in `[.data.frame`, but
     be cautious with `NA`.
   - character: matched to the first column of x's key
   - expression: evaluated within the frame of the data.table
   - data.table: x must have a key. i is _joined_ to x using x's key and the rows 
     in x that match are returned. The number of join columns is determined by 
     min(length(key(x)), if (haskey(i)) length(key(i)) else ncol(i))
 
- j: A single column name, single expresson of column names, list() of expressions of column names, 
     an expression or function call that evaluates to list is expression or list of expressions, 
     evaluated with data.table. _When with=FALSE a vector of names or positions to select_

- by: A single unquoted column name, a list() of expressions of column names, 
      a single character string containing comma separated column names.

**R: `i, j, by`**
**SQL: `WHERE, SELECT, GROUP BY`**

## Cookbook 
### Basic Operations

```r
DF <- ptdemog
DT <- data.table(ptdemog)
DT_mmse <- data.table(mmse)
DT_cogp1 <- data.table(cogp1)
head(DF)
DT
identical(dim(DT), dim(DF))  # TRUE
identical(DF$a, DT$a)  # TRUE
is.list(DF)  # TRUE
is.list(DT)  # TRUE
is.data.frame(DT)  # TRUE

tables()  # see a list of all data.tables in memory 

# We can use data.frame syntax in a data.table, when no keys have been set

DT[DT$RID == 226, ]
DT[2, ]  # 2nd row
DT[2]  # 2nd row
DT[, 5]  # not the 5th column, j is expression so 5 is 5..
DT[, 5, with = FALSE]  # the 5th column, j can be number when with=FALSE, default is TRUE.
colNum = 5
DT[, colNum, with = FALSE]  # same
DT[, PTAGE]  # PTAGE column (as vector)
DT[, list(PTAGE)]  # PTAGE column (as data.table)
DT[, "PTAGE", with = FALSE]  # same
DT[, list(RID, VISCODE, PTAGE)]  # choose multiple columns

setkey(DT, RID)  # DT will be sorted with key
setkeyv(DT, "RID")  # DT will be sorted with key
setkey(DT_cogp1, RID, VISCODE)  # set key with multiple columns, Sorts the table by RID then VISCODE 
setkeyv(DT_cogp1, c("RID", "VISCODE"))
key(DT)
DT[226]  # row with RID == 226, not 226th row
DT[226, ]  # same 
DT[NA]  # not recycle to match the number of rows
DT[NA, ]  # same 
DF[NA, ]  # recycle to match the number of rows
DT[nrow(DT) + 1, ]  # index i out of range
DF[nrow(DF) + 1, ]  # same behavior

DT[DT$RID == 226, ]  # vector scan (slow) but same
DT_cogp1[list(226)]  # rows with RID == 226, binary search (fast)
DT_cogp1[J(226)]  # same 
DT_cogp1["226"]  # same 
DT_cogp1[226]  # Be careful, it's not rows with RID == 226
tables()

DT[2:5, mean(PTAGE)]  # sum(PTAGE) over rows 2 and 3
DT[2:5, paste0(RID, "\n")]  # just for j's side effect
DT[c(FALSE, TRUE)]  # even rows (usual recycling)
# COT2SCOR, COT3SCOR,
DT_cogp1[, mean(COT1SCOR, na.rm = TRUE), by = list(RID, VISCODE)]  # keyed by
DT[, sum(v), by = key(DT)]  # same
DT[, sum(v), by = y]  # ad hoc by

DT[, `:=`(PTAGE_n, scale(PTAGE, center = TRUE, scale = TRUE))]  #vanilla update note the := operator 
DT[, `:=`(PTAGE_c = mean(PTAGE, na.rm = TRUE), PTAGE_s = PTAGE_n - 1)]  #update several columns at once
DT[, `:=`(c("USERID", "USERDATE", "USERID2", "USERDATE2"), NULL)]  #remove several columns at once

```


## Merge and Join
- X[Y, nomatch=NA]: all rows in Y, *right outer join* (default) X[Y]
  - merge(X, Y, all.y=TRUE)
- X[Y, nomatch=0]: only rows with matches in both X and Y, *inner join*
  - merge(X, Y, by=key)
- Y[X]: all rows in X, *left outer join*
  - merge(X, Y, all.x=TRUE)
- unique_keys <- unique(c(X[,t], Y[,t])); Y[X[J(unique_keys)]], all rows from both X and Y - full outer join 
- or X[Y[J(unique_keys)]] 
  - merge(X, Y, all=TRUE) 


```r
# from data.table examples
DF <- data.frame(x = rep(c("a", "b", "c"), each = 3), y = c(1, 3, 6), v = 1:9)
DT <- data.table(x = rep(c("a", "b", "c"), each = 3), y = c(1, 3, 6), v = 1:9, 
    key = "x")
DT
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
## 4: b 1 4
## 5: b 3 5
## 6: b 6 6
## 7: c 1 7
## 8: c 3 8
## 9: c 6 9
```

```r
X <- data.table(c("b", "c"), foo = c(4, 2))
X
```

```
##    V1 foo
## 1:  b   4
## 2:  c   2
```

```r
DT["a", sum(v)]  # j for one group
```

```
##    x V1
## 1: a  6
```

```r
DT[c("a", "b"), sum(v)]  # j for two groups
```

```
##    x V1
## 1: a  6
## 2: b 15
```

```r

DT[X]  # join 
```

```
##    x y v foo
## 1: b 1 4   4
## 2: b 3 5   4
## 3: b 6 6   4
## 4: c 1 7   2
## 5: c 3 8   2
## 6: c 6 9   2
```

```r
DT[X, sum(v)]  # join and eval j for each row in i
```

```
##    x V1
## 1: b 15
## 2: c 24
```

```r
DT[X, mult = "first"]  # first row of each group
```

```
##    x y v foo
## 1: b 1 4   4
## 2: c 1 7   2
```

```r
DT[X, mult = "last"]  # last row of each group
```

```
##    x y v foo
## 1: b 6 6   4
## 2: c 6 9   2
```

```r
DT[X, sum(v) * foo]  # join inherited scope
```

```
##    x V1
## 1: b 60
## 2: c 48
```

```r

setkey(DT, x, y)  # 2-column key
setkeyv(DT, c("x", "y"))  # same

DT["a"]  # join to 1st column of key
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[J("a")]  # same. J() stands for Join, an alias for list()
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[list("a")]  # same
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[.("a")]  # same. In the style of package plyr.
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[J("a", 3)]  # join to 2 columns
```

```
##    x y v
## 1: a 3 2
```

```r
DT[.("a", 3)]  # same
```

```
##    x y v
## 1: a 3 2
```

```r
DT[J("a", 3:6)]  # join 4 rows (2 missing)
```

```
##    x y  v
## 1: a 3  2
## 2: a 4 NA
## 3: a 5 NA
## 4: a 6  3
```

```r
DT[J("a", 3:6), nomatch = 0]  # remove missing
```

```
##    x y v
## 1: a 3 2
## 2: a 6 3
```

```r
DT[J("a", 3:6), roll = TRUE]  # rolling join (locf)
```

```
##    x y v
## 1: a 3 2
## 2: a 4 2
## 3: a 5 2
## 4: a 6 3
```

```r

DT[, sum(v), by = list(y%%2)]  # by expression
```

```
##    y V1
## 1: 1 27
## 2: 0 18
```

```r
DT[, .SD[2], by = x]  # 2nd row of each group
```

```
##    x y v
## 1: a 3 2
## 2: b 3 5
## 3: c 3 8
```

```r
DT[, tail(.SD, 2), by = x]  # last 2 rows of each group
```

```
##    x y v
## 1: a 3 2
## 2: a 6 3
## 3: b 3 5
## 4: b 6 6
## 5: c 3 8
## 6: c 6 9
```

```r
DT[, lapply(.SD, sum), by = x]  # apply through columns by group
```

```
##    x  y  v
## 1: a 10  6
## 2: b 10 15
## 3: c 10 24
```

```r

DT[, list(MySum = sum(v), MyMin = min(v), MyMax = max(v)), by = list(x, y%%2)]  # by 2 expressions
```

```
##    x y MySum MyMin MyMax
## 1: a 1     3     1     2
## 2: a 0     3     3     3
## 3: b 1     9     4     5
## 4: b 0     6     6     6
## 5: c 1    15     7     8
## 6: c 0     9     9     9
```

```r

DT[, sum(v), x][V1 < 20]  # compound query
```

```
##    x V1
## 1: a  6
## 2: b 15
```

```r
DT[, sum(v), x][order(-V1)]  # ordering results
```

```
##    x V1
## 1: c 24
## 2: b 15
## 3: a  6
```

```r

print(DT[, `:=`(z, 42L)])  # add new column by reference
```

```
##    x y v  z
## 1: a 1 1 42
## 2: a 3 2 42
## 3: a 6 3 42
## 4: b 1 4 42
## 5: b 3 5 42
## 6: b 6 6 42
## 7: c 1 7 42
## 8: c 3 8 42
## 9: c 6 9 42
```

```r
print(DT[, `:=`(z, NULL)])  # remove column by reference
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
## 4: b 1 4
## 5: b 3 5
## 6: b 6 6
## 7: c 1 7
## 8: c 3 8
## 9: c 6 9
```

```r
print(DT["a", `:=`(v, 42L)])  # subassign to existing v column by reference
```

```
##    x y  v
## 1: a 1 42
## 2: a 3 42
## 3: a 6 42
## 4: b 1  4
## 5: b 3  5
## 6: b 6  6
## 7: c 1  7
## 8: c 3  8
## 9: c 6  9
```

```r
print(DT["b", `:=`(v2, 84L)])  # subassign to new column by reference (NA padded)
```

```
##    x y  v v2
## 1: a 1 42 NA
## 2: a 3 42 NA
## 3: a 6 42 NA
## 4: b 1  4 84
## 5: b 3  5 84
## 6: b 6  6 84
## 7: c 1  7 NA
## 8: c 3  8 NA
## 9: c 6  9 NA
```

```r

DT[, `:=`(m, mean(v)), by = x][]  # add new column by reference by group
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: a 3 42 NA 42
## 3: a 6 42 NA 42
## 4: b 1  4 84  5
## 5: b 3  5 84  5
## 6: b 6  6 84  5
## 7: c 1  7 NA  8
## 8: c 3  8 NA  8
## 9: c 6  9 NA  8
```

```r
# NB: postfix [] is shortcut to print()

DT[, .SD[which.min(v)], by = x][]  # nested query by group, .SD - Subset of Data.table
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: b 1  4 84  5
## 3: c 1  7 NA  8
```

```r

DT[!J("a")]  # not join
```

```
##    x y v v2 m
## 1: b 1 4 84 5
## 2: b 3 5 84 5
## 3: b 6 6 84 5
## 4: c 1 7 NA 8
## 5: c 3 8 NA 8
## 6: c 6 9 NA 8
```

```r
DT[!"a"]  # same
```

```
##    x y v v2 m
## 1: b 1 4 84 5
## 2: b 3 5 84 5
## 3: b 6 6 84 5
## 4: c 1 7 NA 8
## 5: c 3 8 NA 8
## 6: c 6 9 NA 8
```

```r
DT[!2:4]  # all rows other than 2:4
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: b 3  5 84  5
## 3: b 6  6 84  5
## 4: c 1  7 NA  8
## 5: c 3  8 NA  8
## 6: c 6  9 NA  8
```

```r
DT[x != "b" | y != 3]  # multiple vector scanning approach, slow
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: a 3 42 NA 42
## 3: a 6 42 NA 42
## 4: b 1  4 84  5
## 5: b 6  6 84  5
## 6: c 1  7 NA  8
## 7: c 3  8 NA  8
## 8: c 6  9 NA  8
```

```r
DT[!J("b", 3)]  # same result but much faster
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: a 3 42 NA 42
## 3: a 6 42 NA 42
## 4: b 1  4 84  5
## 5: b 6  6 84  5
## 6: c 1  7 NA  8
## 7: c 3  8 NA  8
## 8: c 6  9 NA  8
```




```r
DT_mmse = data.table(mmse)
DT_cogp1 = data.table(cogp1)
DT_demog = data.table(ptdemog)
setkey(DT_demog, RID)
setkey(DT_cogp1, RID, VISCODE)  # set key with multiple columns, Sorts the table by RID then VISCODE 
setkey(DT_mmse, RID, VISCODE)  # set key with multiple columns, Sorts the table by RID then VISCODE 

# add a new columns treatment
DT_demog[, `:=`(treatment, ifelse(sample(c(0, 1), nrow(DT_demog), replace = TRUE), 
    "treatment", "placebo"))]
```

```
##      RID SITEID VISCODE   USERID   USERDATE    USERID2  USERDATE2 EXAMINIT
##   1:  11    112      sc JUDICKSO 2009-01-16         NA       <NA>      JDE
##   2:  12    202      sc CMCADAMS 2008-02-14         NA       <NA>      JDR
##   3:  18     98      sc   GEMILY 2008-07-08         NA 2011-07-07      NAJ
##   4:  19     86      sc  LMURRAY 2007-12-03         NA       <NA>      SAS
##   5:  20     85      sc SAUCEDAC 2008-10-13         NA       <NA>      N-S
##  ---                                                                      
## 706: 792     94      sc SAUCEDAC 2008-06-15         NA       <NA>      tmj
## 707: 794    212      sc LGORDINE 2009-01-22         NA       <NA>      meg
## 708: 796     83      sc CMCADAMS 2008-10-06         NA       <NA>      N-S
## 709: 798    202      sc    SSAMI 2009-01-07         NA       <NA>      SAC
## 710: 799    113      sc ELVANHOO 2009-02-13 TAM_BU_EDU       <NA>      ASM
##        EXAMDATE PTAGE90 PTAGE PTGENDER                PTETHCAT
##   1: 2008-01-04      No    80   Female Not Hispanic nor Latino
##   2: 2007-12-03      No    81     Male Not Hispanic nor Latino
##   3: 2008-03-25      No    76   Female Not Hispanic nor Latino
##   4: 2008-06-23      No    77   Female Not Hispanic nor Latino
##   5: 2008-12-10      No    76     Male Not Hispanic nor Latino
##  ---                                                          
## 706: 2008-06-26      No    81     Male Not Hispanic nor Latino
## 707: 2008-04-30      No    86   Female Not Hispanic nor Latino
## 708: 2008-03-11      No    82   Female Not Hispanic nor Latino
## 709: 2007-12-17      No    85   Female Not Hispanic nor Latino
## 710: 2008-03-28      No    82   Female Not Hispanic nor Latino
##                       PTRACCAT  PTMARRY PTEDUCAT PTWORKHS
##   1: Black or African American  Married       16       NA
##   2:                     White Divorced       18       NA
##   3:                     White  Widowed       12       NA
##   4:                     White  Married       12       NA
##   5:                     White  Widowed       10      Yes
##  ---                                                     
## 706:                     White  Married       18       NA
## 707:                     White Divorced       16       NA
## 708:                     White  Married       15       NA
## 709:                     White  Married       16       NA
## 710:                     White  Married       14       NA
##                                           PTWORK
##   1:                                     Teacher
##   2:                             Psychotherapist
##   3:          U.S. Senate, Legislative Assistant
##   4:                                   dietician
##   5:                                    Clerical
##  ---                                            
## 706:                                   homemaker
## 707: general practice physician, general surgery
## 708:                    administrative assistant
## 709:                                   housewife
## 710:                                 Social work
##                          PTWRECNT PTNOTRT     PTRTYR               PTHOME
##   1:                      Teacher     Yes 1993-07-15  Condo/Co-op (owned)
##   2:               office manager     Yes 1991-12-31 Retirement Community
##   3:                    Counselor     Yes 1982-07-15                House
##   4:                      RETIRED     Yes 1973-07-15 Retirement Community
##   5:            Gift Store Teller     Yes 1978-07-15  Condo/Co-op (owned)
##  ---                                                                     
## 706:                    Housewife     Yes       <NA>   Apartment (rented)
## 707:                    Homemaker     Yes 1982-07-15                House
## 708:                 psychologist     Yes 1990-07-15                House
## 709:                    housewife      No       <NA>                House
## 710: Computer Research Supervisor      No 1970-07-15                House
##      PTOTHOME PTTLANG PTNLANG    PTOTHNL PTPLANG PTOTHPL treatment
##   1:       NA English English         NA English      NA   placebo
##   2:       NA English English         NA English      NA   placebo
##   3:       NA English English         NA English      NA treatment
##   4:       NA English English         NA English      NA treatment
##   5:       NA English English Portuguese English      NA treatment
##  ---                                                              
## 706:       NA English English         NA English      NA treatment
## 707:       NA English English         NA English      NA   placebo
## 708:       NA English English         NA English      NA   placebo
## 709:       NA English English         NA Spanish      NA treatment
## 710:       NA English English     German English      NA treatment
```

```r
COTs <- colnames(DT_cogp1)[grepl("COT\\d+SCOR", colnames(DT_cogp1), perl = TRUE)]
cols <- c("RID", "VISCODE", COTs)
DT_join <- DT_demog[, c("RID", "treatment", "PTAGE"), with = FALSE][DT_cogp1[, 
    cols, with = FALSE]]
DT_merge <- merge(DT_demog[, c("RID", "treatment", "PTAGE"), with = FALSE], 
    DT_cogp1[, cols, with = FALSE], by = "RID", all.y = TRUE)
identical(DT_join, DT_merge)
```

```
## [1] TRUE
```

```r

DT_join[, lapply(.SD[, COTs, with = FALSE], function(x) mean(x, na.rm = TRUE)), 
    by = "treatment"]  # not preferred 
```

```
##    treatment COT1SCOR COT2SCOR COT3SCOR
## 1:   placebo    7.004    7.735    8.068
## 2: treatment    7.083    7.717    8.026
```

```r
DT_join[, lapply(.SD, function(x) mean(x, na.rm = TRUE)), by = "treatment", 
    .SDcols = COTs]  # preferred
```

```
##    treatment COT1SCOR COT2SCOR COT3SCOR
## 1:   placebo    7.004    7.735    8.068
## 2: treatment    7.083    7.717    8.026
```

```r
DT_join[, lapply(.SD, function(x) mean(x, na.rm = TRUE)), by = "VISCODE", .SDcols = COTs]
```

```
##     VISCODE COT1SCOR COT2SCOR COT3SCOR
##  1:      bl    7.165    7.940    7.993
##  2:     m01    6.608    7.570    7.829
##  3:     m03    7.139    7.832    8.205
##  4:     m07    7.474    7.413    7.961
##  5:     m09    7.068    7.548    8.073
##  6:     m12    6.873    7.633    7.971
##  7:     m15    7.179    7.860    8.061
##  8:     m21    6.822    7.778    8.115
##  9:     m25    6.931    7.404    8.083
## 10:     m26    6.759    7.807    7.741
## 11:     m39    7.168    7.740    8.257
## 12:     m43    7.268    7.721    8.119
## 13:     m45    7.359    7.543    7.865
## 14:    m48e    7.095    7.662    8.162
## 15:     m08    7.173    7.787    7.987
## 16:     m02    7.064    8.037    8.137
## 17:     m11    7.225    7.100    7.857
## 18:     m13    7.394    8.185    7.288
## 19:     m36    7.062    7.784    8.060
## 20:     m47    7.154    7.692    8.289
## 21:     m34    7.400    7.659    7.739
## 22:     m06    7.024    7.585    8.085
## 23:     m19    7.164    7.855    8.082
## 24:     m30    7.026    7.786    8.080
## 25:     m05    6.623    7.971    8.203
## 26:     m10    6.329    7.577    7.826
## 27:     m42    7.085    7.515    8.147
## 28:     m17    7.108    7.364    7.909
## 29:     m18    7.044    7.648    7.944
## 30:     m27    7.040    7.873    7.980
## 31:     m33    6.977    7.719    8.355
## 32:     m44    7.333    6.881    8.262
## 33:     m32    7.044    7.867    8.043
## 34:     m24    6.945    7.856    7.976
## 35:     m16    6.896    7.632    8.088
## 36:     m14    7.076    7.853    8.231
## 37:     m31    6.480    7.500    8.042
## 38:     m22    7.074    7.339    8.175
## 39:     m29    7.352    6.962    8.255
## 40:     m04    6.833    7.803    8.026
## 41:     m28    7.040    7.820    7.608
## 42:     m41    7.220    8.415    7.690
## 43:     m46    6.675    8.050    8.275
## 44:     m23    7.086    7.607    8.267
## 45:     m37    6.826    7.574    8.044
## 46:     m40    7.667    7.533    8.467
## 47:     m20    6.690    8.052    8.068
## 48:     m35    7.149    7.918    7.694
## 49:     m38    6.640    8.020    7.765
##     VISCODE COT1SCOR COT2SCOR COT3SCOR
```

```r
# mean of each COTs by VISCODE and treatment
DT_join[, lapply(.SD, function(x) mean(x, na.rm = TRUE)), by = "VISCODE,treatment", 
    .SDcols = COTs]
```

```
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
##  1:      bl   placebo    6.909    7.897    8.047
##  2:     m01   placebo    6.262    7.511    7.977
##  3:     m03   placebo    7.144    7.897    8.273
##  4:     m07   placebo    7.686    7.000    7.829
##  5:     m09   placebo    7.028    7.650    7.979
##  6:     m12   placebo    6.915    7.670    7.960
##  7:     m15   placebo    7.090    7.833    8.083
##  8:     m21   placebo    6.559    7.828    8.094
##  9:     m25   placebo    7.000    7.586    8.067
## 10:     m26   placebo    7.619    7.955    7.857
## 11:     m39   placebo    7.010    7.716    8.282
## 12:     m43   placebo    6.852    7.586    8.367
## 13:     m45   placebo    7.494    7.809    7.882
## 14:    m48e   placebo    7.075    7.680    8.034
## 15:     m08 treatment    7.172    7.966    8.000
## 16:    m48e treatment    7.113    7.647    8.277
## 17:     m02 treatment    7.294    7.829    8.059
## 18:     m11 treatment    7.057    7.314    8.029
## 19:     m12 treatment    6.832    7.599    7.983
## 20:     m13 treatment    7.364    7.758    7.485
## 21:     m36 treatment    7.129    7.907    8.227
## 22:     m39 treatment    7.327    7.765    8.233
## 23:     m47 treatment    6.880    7.840    7.792
## 24:     m34 treatment    6.773    7.591    7.957
## 25:     m01 treatment    7.062    7.647    7.636
## 26:     m06 treatment    7.142    7.602    7.994
## 27:     m15 treatment    7.259    7.884    8.041
## 28:     m19 treatment    7.464    7.500    8.300
## 29:     m21 treatment    7.087    7.726    8.135
## 30:     m30 treatment    7.015    7.788    8.180
## 31:      bl treatment    7.399    7.980    7.944
## 32:     m05 treatment    6.762    7.860    8.070
## 33:     m10 treatment    6.425    7.500    7.711
## 34:     m45 treatment    7.248    7.327    7.850
## 35:     m05   placebo    6.407    8.154    8.423
## 36:     m36   placebo    6.981    7.630    7.853
## 37:     m42   placebo    7.081    7.541    8.300
## 38:     m03 treatment    7.133    7.764    8.133
## 39:     m09 treatment    7.103    7.461    8.154
## 40:     m17 treatment    6.939    7.029    7.636
## 41:     m18 treatment    6.971    7.458    7.842
## 42:     m27 treatment    7.144    7.844    7.748
## 43:     m33 treatment    6.974    7.597    8.235
## 44:     m44 treatment    6.760    7.250    8.160
## 45:     m06   placebo    6.900    7.566    8.182
## 46:     m32   placebo    6.862    7.893    8.138
## 47:     m24 treatment    6.732    7.985    7.920
## 48:     m16   placebo    6.818    7.364    8.121
## 49:     m30   placebo    7.043    7.783    7.935
## 50:     m14 treatment    7.529    7.943    8.242
## 51:     m31 treatment    6.571    7.250    8.115
## 52:     m42 treatment    7.090    7.490    7.990
## 53:     m22 treatment    7.242    7.235    8.471
## 54:     m26 treatment    6.270    7.714    7.676
## 55:     m29 treatment    7.190    7.429    7.842
## 56:     m14   placebo    6.594    7.758    8.219
## 57:     m04   placebo    6.943    7.639    8.167
## 58:     m17   placebo    7.281    7.719    8.182
## 59:     m28   placebo    7.000    7.690    7.500
## 60:     m31   placebo    6.364    7.818    7.955
## 61:     m33   placebo    6.980    7.867    8.495
## 62:     m34   placebo    8.000    7.727    7.522
## 63:     m41   placebo    7.000    8.714    7.643
## 64:     m46   placebo    6.250    7.550    7.850
## 65:     m23 treatment    7.143    7.576    8.114
## 66:     m41 treatment    7.321    8.259    7.714
## 67:     m19   placebo    6.909    8.188    7.871
## 68:     m24   placebo    7.137    7.739    8.027
## 69:     m18   placebo    7.123    7.859    8.055
## 70:     m08   placebo    7.174    7.674    7.978
## 71:     m11   placebo    7.389    6.886    7.694
## 72:     m22   placebo    6.810    7.500    7.739
## 73:     m27   placebo    6.957    7.897    8.158
## 74:     m29   placebo    7.455    6.656    8.500
## 75:     m44   placebo    8.176    6.389    8.412
## 76:     m37   placebo    6.529    7.556    8.167
## 77:     m25 treatment    6.862    7.214    8.100
## 78:     m40 treatment    7.875    7.522    8.304
## 79:     m20   placebo    6.821    8.286    8.207
## 80:     m23   placebo    7.000    7.652    8.480
## 81:     m35   placebo    7.125    7.462    7.815
## 82:     m40   placebo    7.429    7.545    8.636
## 83:     m07 treatment    7.293    7.775    8.073
## 84:     m02   placebo    6.886    8.196    8.196
## 85:     m04 treatment    6.730    7.950    7.900
## 86:     m32 treatment    7.375    7.824    7.882
## 87:     m20 treatment    6.567    7.833    7.933
## 88:     m47   placebo    7.643    7.429    9.143
## 89:     m16 treatment    6.971    7.886    8.057
## 90:     m28 treatment    7.095    8.000    7.762
## 91:     m43 treatment    8.071    8.000    7.500
## 92:     m13   placebo    7.424    8.625    7.091
## 93:     m38   placebo    6.143    8.143    7.714
## 94:     m37 treatment    7.000    7.586    7.963
## 95:     m38 treatment    7.000    7.931    7.800
## 96:     m10   placebo    6.212    7.677    7.968
## 97:     m35 treatment    7.174    8.435    7.545
## 98:     m46 treatment    7.100    8.550    8.700
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
```

```r
# total counts by VISCODE and treatment
DT_join[, lapply(.SD, function(x) length(x)), by = "VISCODE,treatment", .SDcols = COTs]
```

```
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
##  1:      bl   placebo      280      280      280
##  2:     m01   placebo       45       45       45
##  3:     m03   placebo      178      178      178
##  4:     m07   placebo       35       35       35
##  5:     m09   placebo      145      145      145
##  6:     m12   placebo      226      226      226
##  7:     m15   placebo      135      135      135
##  8:     m21   placebo      131      131      131
##  9:     m25   placebo       30       30       30
## 10:     m26   placebo       22       22       22
## 11:     m39   placebo      106      106      106
## 12:     m43   placebo       30       30       30
## 13:     m45   placebo       90       90       90
## 14:    m48e   placebo      153      153      153
## 15:     m08 treatment       29       29       29
## 16:    m48e treatment      171      171      171
## 17:     m02 treatment       35       35       35
## 18:     m11 treatment       35       35       35
## 19:     m12 treatment      235      235      235
## 20:     m13 treatment       34       34       34
## 21:     m36 treatment      206      206      206
## 22:     m39 treatment      105      105      105
## 23:     m47 treatment       25       25       25
## 24:     m34 treatment       23       23       23
## 25:     m01 treatment       34       34       34
## 26:     m06 treatment      173      173      173
## 27:     m15 treatment      149      149      149
## 28:     m19 treatment       30       30       30
## 29:     m21 treatment      127      127      127
## 30:     m30 treatment      136      136      136
## 31:      bl treatment      305      305      305
## 32:     m05 treatment       43       43       43
## 33:     m10 treatment       40       40       40
## 34:     m45 treatment      110      110      110
## 35:     m05   placebo       27       27       27
## 36:     m36   placebo      165      165      165
## 37:     m42   placebo      101      101      101
## 38:     m03 treatment      168      168      168
## 39:     m09 treatment      168      168      168
## 40:     m17 treatment       34       34       34
## 41:     m18 treatment      143      143      143
## 42:     m27 treatment      113      113      113
## 43:     m33 treatment      119      119      119
## 44:     m44 treatment       25       25       25
## 45:     m06   placebo      163      163      163
## 46:     m32   placebo       29       29       29
## 47:     m24 treatment      202      202      202
## 48:     m16   placebo       33       33       33
## 49:     m30   placebo       94       94       94
## 50:     m14 treatment       35       35       35
## 51:     m31 treatment       28       28       28
## 52:     m42 treatment      100      100      100
## 53:     m22 treatment       35       35       35
## 54:     m26 treatment       37       37       37
## 55:     m29 treatment       21       21       21
## 56:     m14   placebo       33       33       33
## 57:     m04   placebo       36       36       36
## 58:     m17   placebo       33       33       33
## 59:     m28   placebo       30       30       30
## 60:     m31   placebo       22       22       22
## 61:     m33   placebo      101      101      101
## 62:     m34   placebo       23       23       23
## 63:     m41   placebo       14       14       14
## 64:     m46   placebo       20       20       20
## 65:     m23 treatment       36       36       36
## 66:     m41 treatment       28       28       28
## 67:     m19   placebo       33       33       33
## 68:     m24   placebo      222      222      222
## 69:     m18   placebo      130      130      130
## 70:     m08   placebo       47       47       47
## 71:     m11   placebo       36       36       36
## 72:     m22   placebo       23       23       23
## 73:     m27   placebo      140      140      140
## 74:     m29   placebo       33       33       33
## 75:     m44   placebo       18       18       18
## 76:     m37   placebo       18       18       18
## 77:     m25 treatment       30       30       30
## 78:     m40 treatment       24       24       24
## 79:     m20   placebo       29       29       29
## 80:     m23   placebo       25       25       25
## 81:     m35   placebo       27       27       27
## 82:     m40   placebo       22       22       22
## 83:     m07 treatment       41       41       41
## 84:     m02   placebo       46       46       46
## 85:     m04 treatment       40       40       40
## 86:     m32 treatment       17       17       17
## 87:     m20 treatment       31       31       31
## 88:     m47   placebo       14       14       14
## 89:     m16 treatment       35       35       35
## 90:     m28 treatment       21       21       21
## 91:     m43 treatment       14       14       14
## 92:     m13   placebo       33       33       33
## 93:     m38   placebo       21       21       21
## 94:     m37 treatment       29       29       29
## 95:     m38 treatment       30       30       30
## 96:     m10   placebo       33       33       33
## 97:     m35 treatment       23       23       23
## 98:     m46 treatment       20       20       20
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
```

```r

# non missing total counts by VISCODE and treatment
DT_join[, lapply(.SD, function(x) {
    sum(!is.na(x))
}), by = "VISCODE,treatment", .SDcols = COTs]
```

```
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
##  1:      bl   placebo      275      273      274
##  2:     m01   placebo       42       45       43
##  3:     m03   placebo      174      174      176
##  4:     m07   placebo       35       35       35
##  5:     m09   placebo      144      140      140
##  6:     m12   placebo      223      218      223
##  7:     m15   placebo      133      132      132
##  8:     m21   placebo      127      128      127
##  9:     m25   placebo       29       29       30
## 10:     m26   placebo       21       22       21
## 11:     m39   placebo      104      102      103
## 12:     m43   placebo       27       29       30
## 13:     m45   placebo       89       89       85
## 14:    m48e   placebo      147      150      149
## 15:     m08 treatment       29       29       29
## 16:    m48e treatment      168      167      166
## 17:     m02 treatment       34       35       34
## 18:     m11 treatment       35       35       34
## 19:     m12 treatment      232      232      232
## 20:     m13 treatment       33       33       33
## 21:     m36 treatment      194      204      203
## 22:     m39 treatment      104      102      103
## 23:     m47 treatment       25       25       24
## 24:     m34 treatment       22       22       23
## 25:     m01 treatment       32       34       33
## 26:     m06 treatment      169      171      171
## 27:     m15 treatment      147      147      146
## 28:     m19 treatment       28       30       30
## 29:     m21 treatment      126      124      126
## 30:     m30 treatment      134      132      133
## 31:      bl treatment      301      298      301
## 32:     m05 treatment       42       43       43
## 33:     m10 treatment       40       40       38
## 34:     m45 treatment      109      110      107
## 35:     m05   placebo       27       26       26
## 36:     m36   placebo      161      162      163
## 37:     m42   placebo       99       98      100
## 38:     m03 treatment      165      165      165
## 39:     m09 treatment      165      165      162
## 40:     m17 treatment       33       34       33
## 41:     m18 treatment      140      142      139
## 42:     m27 treatment      111      109      107
## 43:     m33 treatment      116      119      115
## 44:     m44 treatment       25       24       25
## 45:     m06   placebo      160      159      159
## 46:     m32   placebo       29       28       29
## 47:     m24 treatment      198      198      199
## 48:     m16   placebo       33       33       33
## 49:     m30   placebo       93       92       93
## 50:     m14 treatment       34       35       33
## 51:     m31 treatment       28       28       26
## 52:     m42 treatment      100      100       97
## 53:     m22 treatment       33       34       34
## 54:     m26 treatment       37       35       37
## 55:     m29 treatment       21       21       19
## 56:     m14   placebo       32       33       32
## 57:     m04   placebo       35       36       36
## 58:     m17   placebo       32       32       33
## 59:     m28   placebo       29       29       30
## 60:     m31   placebo       22       22       22
## 61:     m33   placebo       98       98       99
## 62:     m34   placebo       23       22       23
## 63:     m41   placebo       13       14       14
## 64:     m46   placebo       20       20       20
## 65:     m23 treatment       35       33       35
## 66:     m41 treatment       28       27       28
## 67:     m19   placebo       33       32       31
## 68:     m24   placebo      219      218      219
## 69:     m18   placebo      130      128      128
## 70:     m08   placebo       46       46       46
## 71:     m11   placebo       36       35       36
## 72:     m22   placebo       21       22       23
## 73:     m27   placebo      140      136      139
## 74:     m29   placebo       33       32       32
## 75:     m44   placebo       17       18       17
## 76:     m37   placebo       17       18       18
## 77:     m25 treatment       29       28       30
## 78:     m40 treatment       24       23       23
## 79:     m20   placebo       28       28       29
## 80:     m23   placebo       23       23       25
## 81:     m35   placebo       24       26       27
## 82:     m40   placebo       21       22       22
## 83:     m07 treatment       41       40       41
## 84:     m02   placebo       44       46       46
## 85:     m04 treatment       37       40       40
## 86:     m32 treatment       16       17       17
## 87:     m20 treatment       30       30       30
## 88:     m47   placebo       14       14       14
## 89:     m16 treatment       34       35       35
## 90:     m28 treatment       21       21       21
## 91:     m43 treatment       14       14       12
## 92:     m13   placebo       33       32       33
## 93:     m38   placebo       21       21       21
## 94:     m37 treatment       29       29       27
## 95:     m38 treatment       29       29       30
## 96:     m10   placebo       33       31       31
## 97:     m35 treatment       23       23       22
## 98:     m46 treatment       20       20       20
##     VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
```

```r
# the output is differnt
DT_join[, list(lapply(.SD, function(x) {
    sum(!is.na(x))
})), by = "VISCODE,treatment", .SDcols = COTs]
```

```
##      VISCODE treatment  V1
##   1:      bl   placebo 275
##   2:      bl   placebo 273
##   3:      bl   placebo 274
##   4:     m01   placebo  42
##   5:     m01   placebo  45
##  ---                      
## 290:     m35 treatment  23
## 291:     m35 treatment  22
## 292:     m46 treatment  20
## 293:     m46 treatment  20
## 294:     m46 treatment  20
```

```r

# calculate both missing and non missing total counts by VISCODE and
# treatment
DT_join[, lapply(.SD, function(x) {
    list(sum(!is.na(x)), sum(is.na(x)))
}), by = "VISCODE,treatment", .SDcols = COTs]
```

```
##      VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
##   1:      bl   placebo      275      273      274
##   2:      bl   placebo        5        7        6
##   3:     m01   placebo       42       45       43
##   4:     m01   placebo        3        0        2
##   5:     m03   placebo      174      174      176
##  ---                                             
## 192:     m10   placebo        0        2        2
## 193:     m35 treatment       23       23       22
## 194:     m35 treatment        0        0        1
## 195:     m46 treatment       20       20       20
## 196:     m46 treatment        0        0        0
```

```r
DT_join[, list(lapply(.SD, function(x) {
    sum(!is.na(x))
}), lapply(.SD, function(x) {
    sum(is.na(x))
})), by = "VISCODE,treatment", .SDcols = COTs, verbose = TRUE]
```

```
## Finding groups (bysameorder=FALSE) ... done in 0secs. bysameorder=FALSE and o__ is length 7172
## lapply optimization is on, j unchanged as 'list(lapply(.SD, function(x) {    sum(!is.na(x))}), lapply(.SD, function(x) {    sum(is.na(x))}))'
## GForce is on, left j unchanged
## Old mean optimization is on, left j unchanged.
## Starting dogroups ... Column 1 of j is a named vector (each item down the rows is named, somehow). Please remove those names for efficiency (to save creating them over and over for each group). They are ignored anyway.Column 2 of j is a named vector (each item down the rows is named, somehow). Please remove those names for efficiency (to save creating them over and over for each group). They are ignored anyway.
##   collecting ad hoc groups took 0.000s for 98 calls
##   eval(j) took 0.012s for 98 calls
## done dogroups in 0.012 secs
```

```
##      VISCODE treatment  V1 V2
##   1:      bl   placebo 275  5
##   2:      bl   placebo 273  7
##   3:      bl   placebo 274  6
##   4:     m01   placebo  42  3
##   5:     m01   placebo  45  0
##  ---                         
## 290:     m35 treatment  23  0
## 291:     m35 treatment  22  1
## 292:     m46 treatment  20  0
## 293:     m46 treatment  20  0
## 294:     m46 treatment  20  0
```


## Reference
-[data.table R forge] (http://datatable.r-forge.r-project.org/)
-[stackoverflow for data.table] (http://stackoverflow.com/questions/tagged/data.table)
-[data.table wiki] (http://rwiki.sciviews.org/doku.php?id=packages:cran:data.table)


