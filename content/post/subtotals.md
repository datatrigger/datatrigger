---
title: "Adding totals and subtotals rows with pandas or the tidyverse"
summary: "When dealing with a dataframe, generating aggregate data is a very common task. In my experience, presenting the summary statistics for the whole population or for subgroups directly in the dataframe can be useful, if not necessary. Today, I present my recipe to achieve this with the pandas and tidyverse packages."
date: 2020-10-18
tags: ["python", "r", "pandas", "tidyverse", "total row", "subtotals", "aggregate", "groupby"]
draft: false
---

## An easy problem without a simple solution

Today, we are definitely not talking about complex Data Science. We have a dataset like this one:  
  
|    | Group   | Subgroup   |   Value |
|---:|:--------|:-----------|--------:|
|  0 | X       | a          |       6 |
|  1 | X       | a          |      19 |
|  2 | X       | a          |      14 |
|  3 | X       | z          |      10 |
|  4 | Y       | a          |       7 |
|  5 | Y       | a          |       6 |
|  6 | Y       | a          |      18 |
|  7 | Y       | z          |      10 |
|  8 | Y       | z          |      10 |
|  9 | Y       | z          |       3 |
  
And we would like to programmatically generate a summary, like this one:  
  
|    | Group   | Subgroup   |    Mean of Value |
|---:|:--------|:-----------|---------:|
|  0 | X       | a          | 13       |
|  1 | X       | z          | 10       |
|  2 | X       | Total      | 12.25    |
|  3 | Y       | a          | 10.3333  |
|  4 | Y       | z          |  7.66667 |
|  5 | Y       | Total      |  9       |
|  6 | Total   | Total      | 10.3     |
  

It may seem like something very easy to achieve. Well, I was the first to be surprised when I realized that there is currently no trivial way to do this with both pandas and tidyverse libraries. Even the almighty *Stack Overflow* does not give a clear answer about how to do this, as far as I know. Let us see how to proceed.

## HOW TO

The first idea that comes to mind is to compute the "Total" row on the side, and to bind it to the original dataset. Unfortunately, this works if the total is needed only for the whole population and not for subgroups. Indeed, this method is not appropriate for subtotals because reordering the rows after the concatenation is a real pain.  
To get around this problem, we concatenate the original dataset to itself as much as needeed, after having made "Total" a new level for the features we want to include in the aggregate data.  
  
#### Pandas  

We will work on the following dataset, presented above :  

```python
import pandas as pd
import numpy as np

np.random.seed(42)

df = pd.DataFrame({
"Group": 4*['X']+6*['Y'],
"Subgroup": 3*['a']+['z']+3*['a']+3*['z'],
"Value": np.random.randint(20, size=10)
})
```

###### Concatenate
  
We build the two following datasets :  

```python
df.assign(Subgroup=lambda x: "Total")
```

&nbsp;
|    | Group   | Subgroup   |   Value |
|---:|:--------|:-----------|--------:|
|  0 | X       | Total      |       6 |
|  1 | X       | Total      |      19 |
|  2 | X       | Total      |      14 |
|  3 | X       | Total      |      10 |
|  4 | Y       | Total      |       7 |
|  5 | Y       | Total      |       6 |
|  6 | Y       | Total      |      18 |
|  7 | Y       | Total      |      10 |
|  8 | Y       | Total      |      10 |
|  9 | Y       | Total      |       3 |

and

```python
df.assign(Group=lambda x: "Total", Subgroup=lambda x: "Total")
```

&nbsp;
|    | Group   | Subgroup   |   Value |
|---:|:--------|:-----------|--------:|
|  0 | Total   | Total      |       6 |
|  1 | Total   | Total      |      19 |
|  2 | Total   | Total      |      14 |
|  3 | Total   | Total      |      10 |
|  4 | Total   | Total      |       7 |
|  5 | Total   | Total      |       6 |
|  6 | Total   | Total      |      18 |
|  7 | Total   | Total      |      10 |
|  8 | Total   | Total      |      10 |
|  9 | Total   | Total      |       3 |

Then we concatenate these dataframes to the original dataset with ```pandas.concat()``` :  

```python
df_subtotal = pd.concat([
    df,
    df.assign(Subgroup=lambda x: "Total"),
    df.assign(Group=lambda x: "Total", Subgroup=lambda x: "Total")
])
```

Let us compute the mean for each group/subgroup :  

```python
pd.concat([
    df,
    df.assign(Subgroup=lambda x: "Total"),
    df.assign(Group=lambda x: "Total", Subgroup=lambda x: "Total")
]).groupby(by=['Group', 'Subgroup'], observed=True).mean()
```

&nbsp;
|    | Group   | Subgroup   |    Value |
|---:|:--------|:-----------|---------:|
|  0 | Total   | Total      | 10.3     |
|  1 | X       | Total      | 12.25    |
|  2 | X       | a          | 13       |
|  3 | X       | z          | 10       |
|  4 | Y       | Total      |  9       |
|  5 | Y       | a          | 10.3333  |
|  6 | Y       | z          |  7.66667 |

The results are there, but the order of the rows is clearly not satisfying.  

###### Order levels

To circumvent this issue, we cast the grouping variables as dtype *category*. Calling ```pandas.Series.astype("category")``` is not good enough because by default, categories are unordered. Yet, we want the category "Total" to be the last. We use instances of ```CategoricalDtype``` for this purpose. Get more information [here](https://pandas.pydata.org/pandas-docs/stable/user_guide/categorical.html).  

```python
cat_Group = CategoricalDtype(categories=list(df['Group'].unique()) + ['Total'], ordered=True)
cat_Subgroup = CategoricalDtype(categories=list(df['Subgroup'].unique()) + ['Total'], ordered=True)

df_subtotal['Group'] = df_subtotal['Group'].astype(cat_Group)
df_subtotal['Subgroup'] = df_subtotal['Subgroup'].astype(cat_Subgroup)
```

###### Aggregate

Now the ```groupby()``` command returns the expected result :  

```python
df_subtotal.groupby(by=['Group', 'Subgroup'], observed=True).mean()
```

&nbsp;
|    | Group   | Subgroup   |    Value |
|---:|:--------|:-----------|---------:|
|  0 | X       | a          | 13       |
|  1 | X       | z          | 10       |
|  2 | X       | Total      | 12.25    |
|  3 | Y       | a          | 10.3333  |
|  4 | Y       | z          |  7.66667 |
|  5 | Y       | Total      |  9       |
|  6 | Total   | Total      | 10.3     |

Hooray !
  
## R's Tidyverse

In this case, we use *dplyr* for the concatenation and *forcats* to reorder the levels :  

```r
# Import necessary packages
library(tibble)
library(dplyr)
library(forcats)

# Generate the data
df <- tibble(
  Group = c(rep("X", 4), rep("Y", 6)),
  Subgroup = c(rep("a", 3), "z", rep("a", 3), rep("z", 3)),
  Value = sample(20, 10, replace = TRUE)
)

df %>%
  # Concatenate
  bind_rows(
    df %>% mutate(Subgroup = "Total"),
    df %>% mutate(across(c(Group, Subgroup), ~ "Total"))
  ) %>%
  # Aggregate
  group_by(
    Group,
    Subgroup
  ) %>%
  summarise(
    Mean = mean(Value)
  ) %>%
  ungroup() %>%
  # Reorder factor levels
  mutate(
    Group = Group %>% as.factor() %>% fct_relevel("Total", after = Inf),
    Subgroup = Subgroup %>% as.factor() %>% fct_relevel("Total", after = Inf)
  ) %>%
  # Arrange
  arrange(
    Group,
    Subgroup
  )
  ```

As usual, the full scripts are available on my [GitHub](https://github.com/datatrigger/subtotals).