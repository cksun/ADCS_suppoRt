


#### Table of Contents
[data.table] (#data.table)
   [data.table basic syntax] (#data.table.syntax)
   [data.table cookbook] (#data.table.cookbook)
   [data.table merge & join] (#data.table.merge)
[dplyr] (#dplyr)
   [dplyr basic syntax] (#dplyr.syntax)
   [dplyr cookbook] (#dplyr.cookbook)
   [dplyr merge & join] (#dplyr.merge)

<a name="data.table"/>
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

<a name="data.table.syntax"/>
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

<a name="data.table.cookbook"/>
## Cookbook 
### Basic Operations

```r
DF <- ptdemog 
DT <- data.table(ptdemog)
DT_mmse <- data.table(mmse)
DT_cogp1 <- data.table(cogp1)
head(DF)
DT
identical(dim(DT),dim(DF)) # TRUE
identical(DF$a, DT$a)      # TRUE
is.list(DF)                # TRUE
is.list(DT)                # TRUE
is.data.frame(DT)          # TRUE

tables()                   # see a list of all data.tables in memory 

# We can use data.frame syntax in a data.table, when no keys have been set 

DT[DT$RID == 226, ]               
DT[2, ]                          # 2nd row
DT[2]                            # 2nd row
DT[, 5]                          # not the 5th column, j is expression so 5 is 5..
DT[, 5, with=FALSE]              # the 5th column, j can be number when with=FALSE, default is TRUE.
colNum = 5
DT[, colNum, with=FALSE]         # same
DT[, PTAGE]                      # PTAGE column (as vector)
DT[, list(PTAGE)]                # PTAGE column (as data.table)
DT[, "PTAGE", with=FALSE]        # same
DT[, list(RID, VISCODE, PTAGE)]  # choose multiple columns

setkey(DT, RID)                  # DT will be sorted with key
setkeyv(DT, "RID")               # DT will be sorted with key
setkey(DT_cogp1, RID, VISCODE)   # set key with multiple columns, Sorts the table by RID then VISCODE 
setkeyv(DT_cogp1, c("RID", "VISCODE"))
key(DT)
DT[226]                          # row with RID == 226, not 226th row
DT[226, ]                        # same 
DT[NA]                           # not recycle to match the number of rows
DT[NA, ]                         # same 
DF[NA, ]                         # recycle to match the number of rows
DT[nrow(DT) + 1, ]               # index i out of range
DF[nrow(DF) + 1, ]               # same behavior

DT[DT$RID == 226, ]              # vector scan (slow) but same
DT_cogp1[list(226)]              # rows with RID == 226, binary search (fast)
DT_cogp1[J(226)]                 # same 
DT_cogp1["226"]                  # same 
DT_cogp1[226]                    # Be careful, it's not rows with RID == 226
tables()

DT[2:5, mean(PTAGE)]          # sum(PTAGE) over rows 2 and 3
DT[2:5, paste0(RID,"\n")]     # just for j's side effect
DT[c(FALSE,TRUE)]             # even rows (usual recycling)
# COT2SCOR, COT3SCOR, 
DT_cogp1[, mean(COT1SCOR, na.rm=TRUE), by=list(RID, VISCODE)]   # keyed by
DT[ , sum(v), by=key(DT)]     # same
DT[ , sum(v), by=y]           # ad hoc by

DT[, PTAGE_n:=scale(PTAGE, center = TRUE, scale = TRUE)]         #vanilla update note the := operator 
DT[,`:=`(PTAGE_c=mean(PTAGE, na.rm=TRUE), PTAGE_s=PTAGE_n - 1)]  #update several columns at once
DT[, c("USERID", "USERDATE", "USERID2", "USERDATE2"):=NULL]      #remove several columns at once
```

<a name="data.table.merge"/>
## Merge and Join
- `X[Y, nomatch=NA]`: all rows in Y, *right outer join* (default) X[Y]
  - `merge(X, Y, all.y=TRUE)`
- `X[Y, nomatch=0]`: only rows with matches in both X and Y, *inner join*
  - `merge(X, Y, by=key)`
- `Y[X]`: all rows in X, *left outer join*
  - `merge(X, Y, all.x=TRUE)`
- `unique_keys <- unique(c(X[,t], Y[,t])); Y[X[J(unique_keys)]]`, all rows from both X and Y - full outer join 
- or `X[Y[J(unique_keys)]]` 
  - `merge(X, Y, all=TRUE)`


```r
# from data.table examples
DF <- data.frame(x=rep(c("a","b","c"),each=3), y=c(1,3,6), v=1:9)
DT <- data.table(x=rep(c("a","b","c"),each=3), y=c(1,3,6), v=1:9, key='x')
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
X <- data.table(c("b","c"), foo=c(4,2))
X
```

```
##    V1 foo
## 1:  b   4
## 2:  c   2
```

```r
DT["a", sum(v)]                  # j for one group
```

```
##    x V1
## 1: a  6
```

```r
DT[c("a","b"), sum(v)]           # j for two groups
```

```
##    x V1
## 1: a  6
## 2: b 15
```

```r
DT[X]                            # join 
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
DT[X, sum(v)]                    # join and eval j for each row in i
```

```
##    x V1
## 1: b 15
## 2: c 24
```

```r
DT[X, mult="first"]              # first row of each group
```

```
##    x y v foo
## 1: b 1 4   4
## 2: c 1 7   2
```

```r
DT[X, mult="last"]               # last row of each group
```

```
##    x y v foo
## 1: b 6 6   4
## 2: c 6 9   2
```

```r
DT[X, sum(v)*foo]                # join inherited scope
```

```
##    x V1
## 1: b 60
## 2: c 48
```

```r
setkey(DT,x,y)                   # 2-column key
setkeyv(DT, c("x","y"))           # same

DT["a"]                          # join to 1st column of key
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[J("a")]                       # same. J() stands for Join, an alias for list()
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[list("a")]                    # same
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[.("a")]                       # same. In the style of package plyr.
```

```
##    x y v
## 1: a 1 1
## 2: a 3 2
## 3: a 6 3
```

```r
DT[J("a",3)]                     # join to 2 columns
```

```
##    x y v
## 1: a 3 2
```

```r
DT[.("a",3)]                     # same
```

```
##    x y v
## 1: a 3 2
```

```r
DT[J("a",3:6)]                   # join 4 rows (2 missing)
```

```
##    x y  v
## 1: a 3  2
## 2: a 4 NA
## 3: a 5 NA
## 4: a 6  3
```

```r
DT[J("a",3:6), nomatch=0]        # remove missing
```

```
##    x y v
## 1: a 3 2
## 2: a 6 3
```

```r
DT[J("a",3:6), roll=TRUE]        # rolling join (locf)
```

```
##    x y v
## 1: a 3 2
## 2: a 4 2
## 3: a 5 2
## 4: a 6 3
```

```r
DT[, sum(v), by=list(y%%2)]      # by expression
```

```
##    y V1
## 1: 1 27
## 2: 0 18
```

```r
DT[, .SD[2], by=x]               # 2nd row of each group
```

```
##    x y v
## 1: a 3 2
## 2: b 3 5
## 3: c 3 8
```

```r
DT[, tail(.SD, 2), by=x]         # last 2 rows of each group
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
DT[, lapply(.SD, sum), by=x]     # apply through columns by group
```

```
##    x  y  v
## 1: a 10  6
## 2: b 10 15
## 3: c 10 24
```

```r
DT[, list(MySum=sum(v),
          MyMin=min(v),
          MyMax=max(v)),
by=list(x,y%%2)]                 # by 2 expressions
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
DT[, sum(v), x][V1<20]           # compound query
```

```
##    x V1
## 1: a  6
## 2: b 15
```

```r
DT[, sum(v), x][order(-V1)]      # ordering results
```

```
##    x V1
## 1: c 24
## 2: b 15
## 3: a  6
```

```r
print(DT[, z:=42L])              # add new column by reference
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
print(DT[, z:=NULL])             # remove column by reference
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
print(DT["a", v:=42L])           # subassign to existing v column by reference
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
print(DT["b", v2:=84L])          # subassign to new column by reference (NA padded)
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
DT[,m:=mean(v), by=x][]          # add new column by reference by group
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

DT[,.SD[which.min(v)], by=x][]   # nested query by group, .SD - Subset of Data.table
```

```
##    x y  v v2  m
## 1: a 1 42 NA 42
## 2: b 1  4 84  5
## 3: c 1  7 NA  8
```

```r
DT[!J("a")]                      # not join
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
DT[!"a"]                         # same
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
DT[!2:4]                         # all rows other than 2:4
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
DT[x!="b" | y!=3]                # multiple vector scanning approach, slow
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
DT[!J("b",3)]                    # same result but much faster
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
setkey(DT_cogp1, RID, VISCODE)   # set key with multiple columns, Sorts the table by RID then VISCODE 
setkey(DT_mmse, RID, VISCODE)   # set key with multiple columns, Sorts the table by RID then VISCODE 

# add a new columns treatment
DT_demog[, treatment:= ifelse(sample(c(0, 1), nrow(DT_demog), replace=TRUE), 'treatment', 'placebo')]
```

```
##      RID SITEID PTAGE PTGENDER PTEDUCAT treatment
##   1:  11    112    80   Female       16   placebo
##   2:  12    202    81     Male       18   placebo
##   3:  18     98    76   Female       12 treatment
##   4:  19     86    77   Female       12 treatment
##   5:  20     85    76     Male       10 treatment
##  ---                                             
## 706: 792     94    81     Male       18 treatment
## 707: 794    212    86   Female       16   placebo
## 708: 796     83    82   Female       15   placebo
## 709: 798    202    85   Female       16 treatment
## 710: 799    113    82   Female       14 treatment
```

```r
COTs <- colnames(DT_cogp1)[grepl('COT\\d+SCOR', colnames(DT_cogp1), perl=TRUE)]
cols <- c('RID', 'VISCODE', COTs)
DT_join <- DT_demog[, c('RID', 'treatment', 'PTAGE'), with=FALSE][DT_cogp1[, cols, with=FALSE]]
DT_merge <- merge(DT_demog[, c('RID', 'treatment', 'PTAGE'), with=FALSE], DT_cogp1[, cols, with=FALSE], by='RID', all.y=TRUE)
identical(DT_join, DT_merge)
```

```
## [1] TRUE
```

```r
DT_join[, lapply(.SD[, COTs, with=FALSE], function(x) mean(x, na.rm=TRUE)), by='treatment'] # not preferred 
```

```
##    treatment COT1SCOR COT2SCOR COT3SCOR
## 1:   placebo    7.004    7.735    8.068
## 2: treatment    7.083    7.717    8.026
```

```r
DT_join[, lapply(.SD, function(x) mean(x, na.rm=TRUE)), by='treatment', .SDcols=COTs]       # preferred
```

```
##    treatment COT1SCOR COT2SCOR COT3SCOR
## 1:   placebo    7.004    7.735    8.068
## 2: treatment    7.083    7.717    8.026
```

```r
DT_join[, lapply(.SD, function(x) mean(x, na.rm=TRUE)), by='VISCODE', .SDcols=COTs]
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
head(DT_join[, lapply(.SD, function(x) mean(x, na.rm=TRUE)), by='VISCODE,treatment', .SDcols=COTs])
```

```
##    VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
## 1:      bl   placebo    6.909    7.897    8.047
## 2:     m01   placebo    6.262    7.511    7.977
## 3:     m03   placebo    7.144    7.897    8.273
## 4:     m07   placebo    7.686    7.000    7.829
## 5:     m09   placebo    7.028    7.650    7.979
## 6:     m12   placebo    6.915    7.670    7.960
```

```r
# total counts by VISCODE and treatment 
head(DT_join[, lapply(.SD, function(x) length(x)), by='VISCODE,treatment', .SDcols=COTs])
```

```
##    VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
## 1:      bl   placebo      280      280      280
## 2:     m01   placebo       45       45       45
## 3:     m03   placebo      178      178      178
## 4:     m07   placebo       35       35       35
## 5:     m09   placebo      145      145      145
## 6:     m12   placebo      226      226      226
```

```r
# non missing total counts by VISCODE and treatment 
head(DT_join[, lapply(.SD, function(x) {sum(! is.na(x))}), by='VISCODE,treatment', .SDcols=COTs])
```

```
##    VISCODE treatment COT1SCOR COT2SCOR COT3SCOR
## 1:      bl   placebo      275      273      274
## 2:     m01   placebo       42       45       43
## 3:     m03   placebo      174      174      176
## 4:     m07   placebo       35       35       35
## 5:     m09   placebo      144      140      140
## 6:     m12   placebo      223      218      223
```

```r
# the output is differnt
head(DT_join[, list(lapply(.SD, function(x) {sum(! is.na(x))})), by='VISCODE,treatment', .SDcols=COTs])
```

```
##    VISCODE treatment  V1
## 1:      bl   placebo 275
## 2:      bl   placebo 273
## 3:      bl   placebo 274
## 4:     m01   placebo  42
## 5:     m01   placebo  45
## 6:     m01   placebo  43
```

```r
# calculate both missing and non missing total counts by VISCODE and treatment
# with verbose=TRUE
DT_join[, lapply(.SD, function(x) {
                 list(sum(! is.na(x)), sum(is.na(x)))
          }), by='VISCODE,treatment', .SDcols=COTs]
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
DT_join[, list(lapply(.SD, function(x) {sum(! is.na(x))}), 
               lapply(.SD, function(x) {sum(is.na(x))})), by='VISCODE,treatment', .SDcols=COTs, verbose=TRUE]
```

```
## Finding groups (bysameorder=FALSE) ... done in 0.001secs. bysameorder=FALSE and o__ is length 7172
## lapply optimization is on, j unchanged as 'list(lapply(.SD, function(x) {    sum(!is.na(x))}), lapply(.SD, function(x) {    sum(is.na(x))}))'
## GForce is on, left j unchanged
## Old mean optimization is on, left j unchanged.
## Starting dogroups ... Column 1 of j is a named vector (each item down the rows is named, somehow). Please remove those names for efficiency (to save creating them over and over for each group). They are ignored anyway.Column 2 of j is a named vector (each item down the rows is named, somehow). Please remove those names for efficiency (to save creating them over and over for each group). They are ignored anyway.
##   collecting ad hoc groups took 0.000s for 98 calls
##   eval(j) took 0.011s for 98 calls
## done dogroups in 0.011 secs
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
- [data.table R forge] (http://datatable.r-forge.r-project.org/)
- [data.table Reference manual] (http://cran.r-project.org/web/packages/data.table/data.table.pdf)
- [stackoverflow for data.table] (http://stackoverflow.com/questions/tagged/data.table)
- [data.table wiki] (http://rwiki.sciviews.org/doku.php?id=packages:cran:data.table)

<a name="dplyr"/>
# dplyr

<a name="dplyr.syntax"/>
## Basic syntax
 - use dplyr to transform data
   * `filter(df, , , ...)`: return rows that meet some criteria
   * `select(df, , , ...)`: return subset of columns
     + `starts_with(x)`: names starts with "x"
     + `ends_with(x)`: names ends with "x"
     + `contains(x)`: select all variables whose name contains "x"
     + `matches(x)`: select all variables whose names matches regular expression "x"
     + `num_range("x", 1:5, width=2)`: select all variables from x01 to x05.
     + use `-` to drop variables.
   * `arrage(df, , , ...)`: reorder rows
   * `mutate(df, , , ...)`: add new columns
   * `summarise(df, , , ...)`: reduce each group to a single row.
     + `min(x)`, `median(x)`, `max(x)`, `quantile(x, p)`
     + `n()`, `n_distinct()`, `sum(x)`, `mean(x)`
 - first argument is data.frame and always return data.frame (no drop).

### Pipeline operator
 - `x %>% f(y)` means f(x, y)
 - pronounce `%>%` as then

### do function for general purpose
 - It's slower, but general purpose.
 - `.` represent the current group.

```r
library(dplyr)
library(zoo)
```

```
## Error: there is no package called 'zoo'
```

```r
df <- data.frame(houseID = rep(1:10, each = 10),
                 year = 1995:2004,
                 price = ifelse(runif(10 * 10) > 0.50, NA, exp(rnorm(10 * 10))))
df %>% group_by(houseID) %>% do(na.locf(.))
```

```
## Error: could not find function "na.locf"
```

```r
df %>% group_by(houseID) %>% do(head(., 2))
```

```
## Source: local data frame [20 x 3]
## Groups: houseID
## 
##    houseID year  price
## 1        1 1995 0.3299
## 2        1 1996 2.3244
## 3        2 1995 0.7980
## 4        2 1996     NA
## 5        3 1995     NA
## 6        3 1996 0.9101
## 7        4 1995 0.5766
## 8        4 1996     NA
## 9        5 1995 2.7551
## 10       5 1996     NA
## 11       6 1995     NA
## 12       6 1996 0.4188
## 13       7 1995 0.1309
## 14       7 1996 2.8830
## 15       8 1995     NA
## 16       8 1996     NA
## 17       9 1995     NA
## 18       9 1996     NA
## 19      10 1995     NA
## 20      10 1996     NA
```

```r
df %>% group_by(houseID) %>% do(data.frame(year = .$year[1])) 
```

```
## Source: local data frame [10 x 2]
## Groups: houseID
## 
##    houseID year
## 1        1 1995
## 2        2 1995
## 3        3 1995
## 4        4 1995
## 5        5 1995
## 6        6 1995
## 7        7 1995
## 8        8 1995
## 9        9 1995
## 10      10 1995
```

<a name="dplyr.cookbook"/>
## Cookbook 
### Basic Operations

```r
df <- data.frame(color = c("blue", "black", "blue", "blue", "black"), value = 1:5)
filter(df, color == "blue")
```

```
##   color value
## 1  blue     1
## 2  blue     3
## 3  blue     4
```

```r
filter(df, value %in% c(1, 4))
```

```
##   color value
## 1  blue     1
## 2  blue     4
```

```r
select(df, color)
```

```
##   color
## 1  blue
## 2 black
## 3  blue
## 4  blue
## 5 black
```

```r
select(df, -color)
```

```
##   value
## 1     1
## 2     2
## 3     3
## 4     4
## 5     5
```

```r
arrange(df, color) # order by color with ascending values.
```

```
##   color value
## 1 black     2
## 2 black     5
## 3  blue     1
## 4  blue     3
## 5  blue     4
```

```r
arrange(df, desc(color))
```

```
##   color value
## 1  blue     1
## 2  blue     3
## 3  blue     4
## 4 black     2
## 5 black     5
```

```r
mutate(df, double = 2 * value)
```

```
##   color value double
## 1  blue     1      2
## 2 black     2      4
## 3  blue     3      6
## 4  blue     4      8
## 5 black     5     10
```

```r
mutate(df, double = 2 * value, quadruple = 2 * double)
```

```
##   color value double quadruple
## 1  blue     1      2         4
## 2 black     2      4         8
## 3  blue     3      6        12
## 4  blue     4      8        16
## 5 black     5     10        20
```

```r
summarise(df, total = sum(value))
```

```
##   total
## 1    15
```

```r
by_color <- group_by(df, color)
summarise(by_color, total = sum(value))
```

```
## Source: local data frame [2 x 2]
## 
##   color total
## 1 black     7
## 2  blue     8
```

<a name="dplyr.merge"/>
## Merge and Join
 - `inner_join(X, Y)`: Include only rows in both X and Y.
 - `left_join(X, Y)`: Include all of X, and matching rows of Y 
 - `semi_join(X, Y)`: Include rows of X that match Y.
 - `anti_join(X, Y)`: Include rows of X that don't match Y.


```r
x <- data.frame(name = c("John", "Paul", "George", "Ringo", "Stuart", "Pete"),
                instrument = c("guitar", "bass", "guitar", "drums", "bass", "drums"))

y <- data.frame(name = c("John", "Paul", "George", "Ringo", "Brian"),
                band = c("TRUE", "TRUE", "TRUE",  "TRUE", "FALSE"))
inner_join(x, y)
```

```
##     name instrument band
## 1   John     guitar TRUE
## 2   Paul       bass TRUE
## 3 George     guitar TRUE
## 4  Ringo      drums TRUE
```

```r
left_join(x, y)
```

```
##     name instrument band
## 1   John     guitar TRUE
## 2   Paul       bass TRUE
## 3 George     guitar TRUE
## 4  Ringo      drums TRUE
## 5 Stuart       bass <NA>
## 6   Pete      drums <NA>
```

```r
semi_join(x, y)
```

```
##     name instrument
## 1   John     guitar
## 2   Paul       bass
## 3 George     guitar
## 4  Ringo      drums
```

```r
anti_join(x, y)
```

```
##     name instrument
## 1   Pete      drums
## 2 Stuart       bass
```


## Reference
- [useR2014 Hadley's dplyr tutorial] (https://www.dropbox.com/sh/i8qnluwmuieicxc/AAAgt9tIKoIm7WZKIyK25lh6a)
