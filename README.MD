## Introduction

`expss` package provides tabulation functions with support for 'SPSS'-style labels, multiple / nested banners, weights, multiple-response variables and significance testing. There are facilities for nice output of tables in 'knitr', R notebooks, 'Shiny' and 'Jupyter' notebooks. Proper methods for labelled variables add value labels support to base R functions and to some functions from other packages. Additionally, the package offers useful functions for data processing in marketing research / social surveys - popular data transformation functions from 'SPSS' Statistics ('RECODE', 'COUNT', 'COMPUTE', 'DO IF', etc.) and 'Excel' ('COUNTIF', 'VLOOKUP', etc.). Package is intended to help people to move data processing from 'Excel'/'SPSS' to R. See examples below. You can get help about
any function by typing `?function_name` in the R console.

### Links

- [Online introduction - visit it to view code with results](http://gdemin.github.io/expss/)
- [expss on CRAN](https://cran.r-project.org/package=expss)
- [expss on Github](https://github.com/gdemin/expss)
- [Issues](https://github.com/gdemin/expss/issues)

## Installation

`expss` is on CRAN, so for installation you can print in the console
`install.packages("expss")`.

## Cross-tablulation examples

We will use for demonstartion well-known `mtcars` dataset. Let's start with adding labels to the dataset. Then we can continue with tables creation.

```{r, message=FALSE, warning=FALSE}
library(expss)
data(mtcars)
mtcars = apply_labels(mtcars,
                      mpg = "Miles/(US) gallon",
                      cyl = "Number of cylinders",
                      disp = "Displacement (cu.in.)",
                      hp = "Gross horsepower",
                      drat = "Rear axle ratio",
                      wt = "Weight (1000 lbs)",
                      qsec = "1/4 mile time",
                      vs = "Engine",
                      vs = c("V-engine" = 0,
                             "Straight engine" = 1),
                      am = "Transmission",
                      am = c("Automatic" = 0,
                             "Manual"=1),
                      gear = "Number of forward gears",
                      carb = "Number of carburetors"
)

```

For quick cross-tabulation there are `fre` and `cro` family of function. For simplicity we demonstrate here only `cro_cpct` which caluclates column percent. Documentation for other functions, such as `cro_cases` for counts, `cro_rpct` for row percent, `cro_tpct` for table percent and `cro_fun` for custom summary functions can be seen by typing `?cro` and `?cro_fun` in the console. 

```{r}
# 'cro' examples
# Table with multiple banners (column %).
mtcars %>% 
    calculate(cro_cpct(cyl, list(total(), am, vs))) 

# Table with nested banners (column %).          
mtcars %>% 
    calculate(cro_cpct(cyl, list(total(), am %nest% vs)))       

```
We have more sophisticated interface for table construction with `magrittr` piping. Table construction consists of at least of three functions chained with pipe operator: `%>%`. At first we need to specify variables for which statistics will be computed with `tab_cells`. Secondary, we calculate statistics with one of the `tab_stat_*` functions. And last, we finalize table creation with `tab_pivot`, e. g.: `dataset %>% tab_cells(variable) %>% tab_stat_cases() %>% tab_pivot()`. After that we can optionally sort table with `tab_sort_asc`, drop empty rows/columns with `drop_rc` and transpose with `tab_transpose`. Resulting table is just a `data.frame` so we can use usual R operations on it. Detailed documentation for table creation can be seen via `?tables`. For significance testing see `?significance`.
Generally, tables automatically translated to HTML for output in knitr or Jupyter notebooks. However, if we want HTML output in the R notebooks or in the RStudio viewer we need to set options for that: `expss_output_rnotebook()` or `expss_output_viewer()`. 

```{r}
# simple example
mtcars %>% 
    tab_cells(cyl) %>% 
    tab_cols(total(), am) %>% 
    tab_stat_cpct() %>% 
    tab_pivot()

# if we need caption then we use 'htmlTable'
mtcars %>% 
    tab_cells(mpg, disp, hp, wt, qsec) %>%
    tab_cols(total(), am) %>% 
    tab_stat_mean_sd_n() %>%
    tab_last_sig_means(subtable_marks = "both") %>% 
    tab_pivot() %>% 
    htmlTable(caption = "Table with summary statistics and significance marks.")

# Table with the same summary statistics. Statistics labels in columns.
mtcars %>% 
    tab_cells(mpg, disp, hp, wt, qsec) %>%
    tab_cols(total(label = "#Total| |"), am) %>% 
    tab_stat_fun(Mean = w_mean, "Std. dev." = w_sd, "Valid N" = w_n, method = list) %>%
    tab_pivot()

# Different statistics for different variables.
mtcars %>%
    tab_cols(total(), vs) %>%
    tab_cells(mpg) %>% 
    tab_stat_mean() %>% 
    tab_stat_valid_n() %>% 
    tab_cells(am) %>%
    tab_stat_cpct(total_row_position = "none", label = "col %") %>%
    tab_stat_rpct(total_row_position = "none", label = "row %") %>%
    tab_stat_tpct(total_row_position = "none", label = "table %") %>%
    tab_pivot(stat_position = "inside_rows") 

# Table with split by rows and with custom totals.
mtcars %>% 
    tab_cells(cyl) %>% 
    tab_cols(total(), vs) %>% 
    tab_rows(am) %>% 
    tab_stat_cpct(total_row_position = "above",
                  total_label = c("number of cases", "row %"),
                  total_statistic = c("u_cases", "u_rpct")) %>% 
    tab_pivot()

# Linear regression by groups.
mtcars %>% 
    tab_cells(dtfrm(mpg, disp, hp, wt, qsec)) %>% 
    tab_cols(total(label = "#Total| |"), am) %>% 
    tab_stat_fun_df(
        function(x){
            frm = reformulate(".", response = names(x)[1])
            model = lm(frm, data = x)
            dtfrm('Coef.' = coef(model), 
                  confint(model)
            )
        }    
    ) %>% 
    tab_pivot() 
```

## Example of data processing with multiple-response variables

Here we use truncated dataset with data from product test of two samples of
chocolate sweets. 150 respondents tested two kinds of sweets (codenames:
VSX123 and SDF546). Sample was divided into two groups (cells) of 75
respondents in each group. In cell 1 product VSX123 was presented first and
then SDF546. In cell 2 sweets were presented in reversed order. Questions
about respondent impressions about first product are in the block A (and
about second tested product in the block B). At the end of the questionnaire 
there was a question about the preferences between sweets.

List of variables:

- `id` Respondent Id
- `cell` First tested product (cell number)
- `s2a` Age
- `a1_1-a1_6` What did you like in these sweets? Multiple response. First tested product
- `a22` Overall quality. First tested product
- `b1_1-b1_6` What did you like in these sweets? Multiple response. Second tested product
- `b22` Overall quality. Second tested product
- `c1` Preferences

```{r}

data(product_test)

w = product_test # shorter name to save some keystrokes

# here we recode variables from first/second tested product to separate variables for each product according to their cells
# 'h' variables - VSX123 sample, 'p' variables - 'SDF456' sample
# also we recode preferences from first/second product to true names
# for first cell there are no changes, for second cell we should exchange 1 and 2.
w = w %>% 
    do_if(cell == 1, {
        recode(a1_1 %to% a1_6, other ~ copy) %into% (h1_1 %to% h1_6)
        recode(b1_1 %to% b1_6, other ~ copy) %into% (p1_1 %to% p1_6)
        recode(a22, other ~ copy) %into% h22
        recode(b22, other ~ copy) %into% p22
        c1r = c1
    }) %>% 
    do_if(cell == 2, {
        recode(a1_1 %to% a1_6, other ~ copy) %into% (p1_1 %to% p1_6)
        recode(b1_1 %to% b1_6, other ~ copy) %into% (h1_1 %to% h1_6)
        recode(a22, other ~ copy) %into% p22
        recode(b22, other ~ copy) %into% h22
        recode(c1, 1 ~ 2, 2 ~ 1, other ~ copy) %into% c1r
    }) %>% 
    compute({
        # recode age by groups
        age_cat = recode(s2a, lo %thru% 25 ~ 1, lo %thru% hi ~ 2)
        # count number of likes
        # codes 2 and 99 are ignored.
        h_likes = count_row_if(1 | 3 %thru% 98, h1_1 %to% h1_6) 
        p_likes = count_row_if(1 | 3 %thru% 98, p1_1 %to% p1_6) 
    })

# here we prepare labels for future usage
codeframe_likes = num_lab("
    1 Liked everything
    2 Disliked everything
    3 Chocolate
    4 Appearance
    5 Taste
    6 Stuffing
    7 Nuts
    8 Consistency
    98 Other
    99 Hard to answer
")

overall_liking_scale = num_lab("
    1 Extremely poor 
    2 Very poor
    3 Quite poor
    4 Neither good, nor poor
    5 Quite good
    6 Very good
    7 Excellent
")

w = apply_labels(w, 
    c1r = "Preferences",
    c1r = num_lab("
        1 VSX123 
        2 SDF456
        3 Hard to say
    "),
    
    age_cat = "Age",
    age_cat = c("18 - 25" = 1, "26 - 35" = 2),
    
    h1_1 = "Likes. VSX123",
    p1_1 = "Likes. SDF456",
    h1_1 = codeframe_likes,
    p1_1 = codeframe_likes,
    
    h_likes = "Number of likes. VSX123",
    p_likes = "Number of likes. SDF456",
    
    h22 = "Overall quality. VSX123",
    p22 = "Overall quality. SDF456",
    h22 = overall_liking_scale,
    p22 = overall_liking_scale
)

```
Are there any significant differences between preferences? Yes, difference is significant.
```{r}
# 'tab_mis_val(3)' remove 'hard to say' from vector 
w %>% tab_cols(total(), age_cat) %>% 
      tab_cells(c1r) %>% 
      tab_mis_val(3) %>% 
      tab_stat_cases() %>% 
      tab_last_sig_cases() %>% 
      tab_pivot()
    
```
Further we calculate distribution of answers in the survey questions. 
```{r}
# lets specify repeated parts of table creation chains
banner = w %>% tab_cols(total(), age_cat, c1r) 
# column percent with significance
tab_cpct_sig = . %>% tab_stat_cpct() %>% 
                    tab_last_sig_cpct(sig_labels = paste0("<b>",LETTERS, "</b>"))

# means with siginifcance
tab_means_sig = . %>% tab_stat_mean_sd_n(labels = c("<b><u>Mean</u></b>", "sd", "N")) %>% 
                      tab_last_sig_means(
                          sig_labels = paste0("<b>",LETTERS, "</b>"),   
                          keep = "means")

# Preferences
banner %>% 
    tab_cells(c1r) %>% 
    tab_cpct_sig() %>% 
    tab_pivot() 

# Overall liking
banner %>%  
    tab_cells(h22) %>% 
    tab_means_sig() %>% 
    tab_cpct_sig() %>%  
    tab_cells(p22) %>% 
    tab_means_sig() %>% 
    tab_cpct_sig() %>%
    tab_pivot() 

# Likes
banner %>% 
    tab_cells(h_likes) %>% 
    tab_means_sig() %>% 
    tab_cells(mrset(h1_1 %to% h1_6)) %>% 
    tab_cpct_sig() %>% 
    tab_cells(p_likes) %>% 
    tab_means_sig() %>% 
    tab_cells(mrset(p1_1 %to% p1_6)) %>% 
    tab_cpct_sig() %>%
    tab_pivot() 

# below more complciated table were we compare likes side by side
# Likes - side by side comparison
w %>% 
    tab_cols(total(label = "#Total| |"), c1r) %>% 
    tab_cells(list(unvr(mrset(h1_1 %to% h1_6)))) %>% 
    tab_stat_cpct(label = var_lab(h1_1)) %>% 
    tab_cells(list(unvr(mrset(p1_1 %to% p1_6)))) %>% 
    tab_stat_cpct(label = var_lab(p1_1)) %>% 
    tab_pivot(stat_position = "inside_columns") 

```

We can save labelled dataset as *.csv file with accompanying R code for labelling.

```{r, eval=FALSE}
write_labelled_csv(w, file  filename = "product_test.csv")
```

Or, we can save dataset as *.csv file with SPSS syntax to read data and apply labels.

```{r, eval=FALSE}
write_labelled_spss(w, file  filename = "product_test.csv")
```

## Labels support for base R

Here we demonstrate labels support in base R - value labels automatically used as factors levels. So every function which converts labelled variable to factor will utilize labels. 
```{r, fig.height=6, fig.width=7}
library(knitr)
with(mtcars, table(am, vs))

boxplot(mpg ~ am, data = mtcars)
```

There is a special function for variables labels support - `use_labels`. By now variables labels support available only for expression which will be evaluated inside data.frame.
```{r}
# table with dimension names
use_labels(mtcars, table(am, vs)) 

# linear regression
use_labels(mtcars, lm(mpg ~ wt + hp + qsec)) %>% summary
```

And, finally, `ggplot2` graphics with variables and value labels:
```{r, fig.height=6, fig.width=7}
library(ggplot2)
use_labels(mtcars, {
    # 'vars(other)' is needed to get all 'mtcars' data.frame inside expression 
    ggplot(vars(other)) +
        geom_point(aes(y = mpg, x = wt, color = qsec)) +
        facet_grid(am ~ vs)
}) 
```


## Excel functions translation guide

```{r, message=FALSE, warning=FALSE, include=FALSE}
library(expss)
```

Excel toy table:

|       | A   | B   | C    |
|:------|----:|----:|-----:|
| **1** |  2  | 15  |  50  |
| **2** |  1  | 70  |  80  |
| **3** |  3  | 30  |  40  |
| **4** |  2  | 30  |  40  |

Code for creating the same table in R:

```{r, results='hide', message=FALSE, warning=FALSE}
w = read.csv(text = "
a,b,c
2,15,50
1,70,80
3,30,40
2,30,40"
)

```
`w` is the name of our table.

##### IF

**Excel: `IF(B1>60, 1, 0)`**

**R:**
Here we create new column with name `d` with results. `ifelse` function is from base R not from 'expss' package but included here for completeness.
```{r, eval=FALSE}
w$d = ifelse(w$b>60, 1, 0)
```
If we need to use multiple transformations it is often convenient to use `compute` function. Inside `compute` we can put arbitrary number of the statements:

```{r, eval=FALSE}
w = compute(w, {
    d = ifelse(b>60, 1, 0)
    e = 42
    abc_sum = sum_row(a, b, c)
    abc_mean = mean_row(a, b, c)
})
```

##### COUNTIF

Count 1's in the entire dataset.

**Excel: `COUNTIF(A1:C4, 1)`**

**R:**
```{r, eval=FALSE}
count_if(1, w)
```
or
```{r, eval=FALSE}
calculate(w, count_if(1, a, b, c))
```    
Count values greater than 1 in each row of the dataset.

**Excel: `COUNTIF(A1:C1, ">1")`**

**R**: 
```{r, eval=FALSE}
w$d = count_row_if(gt(1), w)  
```
or

```{r, eval=FALSE}
w = compute(w, {
    d = count_row_if(gt(1), a, b, c) 
})

```

Count values less than or equal to 1 in column A of the dataset.

**Excel: `COUNTIF(A1:A4, "<=1")`**

**R**: 
```{r, eval=FALSE}
count_col_if(le(1), w$a)
```

**Table of criteria:**

| Excel   |   R      |
|---------|----------|
| "<1"    |   lt(1)  |
| "<=1"   |   le(1)  |
| "<>1"	  |   ne(1)  |
| "=1"    |   eq(1)  |
| ">=1"   |   ge(1)  |
| ">1"    |   gt(1)  |



##### SUM/AVERAGE
Sum all values in the dataset.

**Excel: `SUM(A1:C4)`**

**R:**
```{r, eval=FALSE}
sum(w, na.rm = TRUE)
```

Calculate average of each row of the dataset.

**Excel: `AVERAGE(A1:C1)`**

**R**: 
```{r, eval=FALSE}
w$d = mean_row(w)  
```
or

```{r, eval=FALSE}
w = compute(w, {
    d = mean_row(a, b, c) 
})

```

Sum values of column `A` of the dataset.

**Excel: `SUM(A1:A4)`**

**R**: 
```{r, eval=FALSE}
sum_col(w$a)
```

##### SUMIF/AVERAGEIF
Sum values greater than 40 in the entire dataset.

**Excel: `SUMIF(A1:C4, ">40")`**

**R:**
```{r, eval=FALSE}
sum_if(gt(40), w)
```
or
```{r, eval=FALSE}
calculate(w, sum_if(gt(40), a, b, c))
```    
Sum values less than 40 in the each row of the dataset.

**Excel: `SUMIF(A1:C1, "<40")`**

**R**: 
```{r, eval=FALSE}
w$d = sum_row_if(lt(40), w)  
```
or

```{r, eval=FALSE}
w = compute(w, {
    d = sum_row_if(lt(40), a, b, c) 
})

```

Calculate average of `B` column with column `A` values less than 3.

**Excel: `AVERAGEIF(A1:A4, "<3", B1:B4)`**

**R**: 
```{r, eval=FALSE}
mean_col_if(lt(3), w$a, data = w$b)
```
or, if we want calculate means for both `b` and `c` columns:
```{r, eval=FALSE}
calculate(w, mean_col_if(lt(3), a, data = dtfrm(b, c)))
```

##### VLOOKUP

Our dictionary for lookup:

|       | X   | Y        |
|:------|----:|:---------|
| **1** |  1  | apples   |
| **2** |  2  | oranges  |
| **3** |  3  | peaches  |


Code for creating the same dictionary in R:

```{r, eval=FALSE}
dict = read.csv(text = "
x,y
1,apples
2,oranges
3,peaches",
stringsAsFactors = FALSE
)
```

**Excel: `VLOOKUP(A1, $X$1:$Y$3, 2, FALSE)`**

**R**: 
```{r, eval=FALSE}
w$d = vlookup(w$a, dict, 2)
```
or, we can use column names:
```{r, eval=FALSE}
w$d = vlookup(w$a, dict, "y")
```

## SPSS functions translation guide

##### COMPUTE

**SPSS**:
```
COMPUTE d = 1.
```

**R**:
```{r, eval=FALSE}
w$d = 1
```
or, in the specific data.frame

```{r, results='hide', message=FALSE, warning=FALSE}
w = compute(w, {
    d = 1
})
```
There can be arbitrary number of statements inside `compute`.

##### IF

**SPSS**:
```
IF(a = 3) d = 2.
```

**R**:

```{r, eval=FALSE}
w = compute(w, {
    d = ifelse(a == 3, 2, NA)
})
```
or,

```{r, eval=FALSE}
w = compute(w, {
    d = ifs(a == 3 ~ 2)
})
```

##### DO IF

**SPSS**:
```
DO IF (a>1).
    COMPUTE d = 4.
END IF.
```

**R**:
```{r, eval=FALSE}
w = do_if(w, a>1, {
    d = 4
})
```

There can be arbitrary number of statements inside `do_if`.

##### COUNT

**SPSS**:
```
COUNT cnt = a1 TO a5 (LO THRU HI).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(lo %thru% hi, a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5 (SYSMIS).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(NA, a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5 (1 THRU 5).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(1 %thru% 5, a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5 (1 THRU HI).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(1 %thru% hi, a1 %to% a5)
```
or,
```{r, eval=FALSE}
cnt = count_row_if(ge(1), a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5 (LO THRU 1).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(lo %thru% 1, a1 %to% a5)
```
or,
```{r, eval=FALSE}
cnt = count_row_if (le(1), a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5 (1 THRU 5, 99).
```
**R**:
```{r, eval=FALSE}
cnt = count_row_if(1 %thru% 5 | 99, a1 %to% a5)
```

**SPSS**:
```
COUNT cnt = a1 TO a5(1,2,3,4,5, SYSMIS).
```

**R**:
```{r, eval=FALSE}
cnt = count_row_if(c(1:5, NA), a1 %to% a5)
```

`count_row_if` can be used inside `compute`.

##### RECODE

**SPSS**:
```
RECODE V1 (0=1) (1=0) (2, 3=-1) (9=9) (ELSE=SYSMIS)
```
**R**:
```{r, eval=FALSE}
recode(v1) = c(0 ~ 1, 1 ~ 0, 2:3 ~ -1, 9 ~ 9, other ~ NA)
```
**SPSS**:
```
RECODE QVAR(1 THRU 5=1)(6 THRU 10=2)(11 THRU HI=3)(ELSE=0).
```
**R**:
```{r, eval=FALSE}
recode(qvar) = c(1 %thru% 5 ~ 1, 6 %thru% 10 ~ 2, 11 %thru% hi ~ 3, other ~ 0)
```
**SPSS**:
```
RECODE STRNGVAR ('A', 'B', 'C'='A')('D', 'E', 'F'='B')(ELSE=' '). 
```
**R**:
```{r, eval=FALSE}
recode(strngvar) = c(c('A', 'B', 'C') ~ 'A', c('D', 'E', 'F') ~ 'B', other ~ ' ')
```
**SPSS**:
```
RECODE AGE (MISSING=9) (18 THRU HI=1) (0 THRU 18=0) INTO VOTER. 
```
**R**:
```{r, eval=FALSE}
voter = recode(age, NA ~ 9, 18 %thru% hi ~ 1, 0 %thru% 18 ~ 0)
# or
recode(age, NA ~ 9, 18 %thru% hi ~ 1, 0 %thru% 18 ~ 0) %into% voter
```

`recode` can be used inside `compute`.

##### VARIABLE LABELS
**SPSS**:
```
VARIABLE LABELS a "Fruits"
                b "Cost"
                c "Price".
```
**R**:
```{r, results='hide'}
w = apply_labels(w,
                 a = "Fruits",
                 b = "Cost",
                 c = "Price"
)
```

##### VALUE LABELS
**SPSS**:
```
VALUE LABELS a
    1 "apples"
    2 "oranges"
    3 "peaches". 
```

**R**:
```{r, results='hide'}
w = apply_labels(w, 
                 a = num_lab("
                        1 apples
                        2 oranges
                        3 peaches 
                    ")
)
```
or, 
```{r, eval=FALSE}
val_lab(w$a) = num_lab("
    1 apples
    2 oranges
    3 peaches 
")

```
##### Tables
**R**:
```{r}
fre(w$a) # Frequency of fruits
cro_cpct(w$b, w$a) # Column percent of cost by fruits
cro_mean(dtfrm(w$b, w$c), w$a) # Mean cost and price by fruits
```


