Getting Data from the Game of Thrones API
================
Rachel Liu
2022-07-15

-   [Setup](#setup)
-   [Introduction](#introduction)
-   [Import and clean the data](#import-and-clean-the-data)
    -   [Scrape Wikipedia for a list of main Game of Thrones characters
        and format
        names](#scrape-wikipedia-for-a-list-of-main-game-of-thrones-characters-and-format-names)
    -   [Obtain data for a single GOT character from APIOIAF with custom
        function](#obtain-data-for-a-single-got-character-from-apioiaf-with-custom-function)
    -   [Use url_getter() to iterate and obtain each GET url of main
        characters](#use-url_getter-to-iterate-and-obtain-each-get-url-of-main-characters)
    -   [Produce tidy data tibble](#produce-tidy-data-tibble)
    -   [Bonus: Produce untidy tibble with
        allegiances](#bonus-produce-untidy-tibble-with-allegiances)
-   [Explore the Data](#explore-the-data)
    -   [Character death bar chart](#character-death-bar-chart)
    -   [Character death by gender](#character-death-by-gender)
    -   [Character death by book
        apperances](#character-death-by-book-apperances)
    -   [Character death by number of book
        POVs](#character-death-by-number-of-book-povs)
    -   [Character death by TV
        apperances](#character-death-by-tv-apperances)
    -   [Summary](#summary)
-   [Machine Learning Modeling](#machine-learning-modeling)
    -   [Logistic Regression model](#logistic-regression-model)
    -   [Logistic Regression model with tuned
        hyperparameters](#logistic-regression-model-with-tuned-hyperparameters)
    -   [Random forest model with tuned
        hyperparameters](#random-forest-model-with-tuned-hyperparameters)
    -   [Last fit: best random forest
        model](#last-fit-best-random-forest-model)

# Setup

``` r
# Set message=FALSE globally
knitr::opts_chunk$set(message = FALSE, warning=FALSE)

# Import necessary libraries
library(tidyverse)
library(httr)
library(jsonlite)
library(rvest)
library(tidymodels)
# install.packages("glmnet")
# install.packages("ranger")
```

# Introduction

Legendary TV show Game of Thrones is infamous for its character deaths.
Let???s look at the main Game of Thrones characters to examine the
relationship between character death and several factors. Does a higher
number of book appearances (out of 5 total GOT books so far) affect
whether or not a character will die? Does having a higher number of
books for in which the character has at least 1 point of view chapter
affect whether or not they will die?

For the night is dark and full of terrors. Let???s get into it.

**Objective**: Examine the relationship between main character death in
the Game of Thrones TV Series as it relates to several factors such as
gender, book appearance, TV appearance, and more.

**Logistics**: Data is obtained from the [an API of Ice and
Fire](https://anapioficeandfire.com/). No authentication is required to
access this API. For more information and documentation, visit the
[APIOIAF documentation](https://anapioficeandfire.com/Documentation).

# Import and clean the data

## Scrape Wikipedia for a list of main Game of Thrones characters and format names

I was reluctant to scrape data from Wikipedia, but I couldn???t find a
list of Game of Thrones main characters anywhere else online, and
working with every character in the API (all 2138 of them) was
definitely too overwhelming.

``` r
# scrape Wikipedia for a list of 43 GOT main characters
got_rank <- read_html(x = "https://en.wikipedia.org/wiki/List_of_Game_of_Thrones_characters")
main_chars <- html_elements(x = got_rank, css = "tbody:nth-child(2) td:nth-child(2)") %>%
    html_text2() 

# Use regex to clean up scraped list of main characters, adjusting nicknames to match API character names
main_chars_mod <- main_chars %>% 
  str_replace(" \".+\"", "") %>% 
  unique() %>% 
  str_replace("The High Sparrow", "High Septon") %>% 
  str_replace("Robert Baratheon", "Robert I Baratheon") %>% 
  str_replace("Bran Stark", "Brandon Stark") %>% 
  str_replace("Tormund Giantsbane", "Tormund") %>% 
  str_replace("Khal ", "") %>% 
  str_replace("Ramsay Bolton", "Ramsay Snow") %>% 
  str_replace_all(" ", "+")
# Eliminate characters from the show that don't appear on APIOIAF
main_chars_mod <- main_chars_mod[!main_chars_mod %in% c("Talisa+Maegyr")]
```

## Obtain data for a single GOT character from APIOIAF with custom function

``` r
# get_allegiance() function: extracts JSON character data about house allegiance from a single url of type chr
get_allegiance <- function(url){
    allegiance <- GET(url = url) %>%
      content(type = "text", encoding = "UTF-8") %>%
      fromJSON()
    return(allegiance$name)
  }

# get_character() function: parses JSON character data from url of type chr for a single character
get_character <- function(url) {
   GET(url) %>% 
    content(type = "text", encoding = "UTF-8") %>% 
    prettify %>% 
    fromJSON(flatten = TRUE) %>% 
    list() 
}

# Test with Jon Snow
jon_snow <- get_character("https://anapioficeandfire.com/api/characters?name=Jon+Snow")
```

## Use url_getter() to iterate and obtain each GET url of main characters

``` r
# url_getter(): iterates over a list of names and obtains the GET urls for each
url_getter <- function(list_names){
  base_url <- "https://www.anapioficeandfire.com/api/characters?"
  
  url_main_chars <- vector(mode = "list", length = length(list_names))
  for (i in 1:length(list_names)) {
    url_main_chars[[i]] <- paste(base_url, "name=", list_names[i], sep = "")
  }
  url_main_chars
}

url_main_chars <- url_getter(main_chars_mod)
```

## Produce tidy data tibble

``` r
max_num <- length(url_main_chars)

# Iterate over character urls using a loop and bind successive df together
got_char <- vector(mode = "list", length = max_num)
got_char_tibble <- tibble()
for (i in 1:max_num) {
  got_char[i] <- get_character(url_main_chars[[i]])
  got_char_tibble <- bind_rows(got_char_tibble, got_char[i])
}

# Mutate relevant variables and clean, store as tidy_got tibble  
base_got <- got_char_tibble %>% 
  mutate(
    alive = if_else(died == "", TRUE, FALSE),
    book_appearance_times = lengths(books),
    book_pov_times = lengths(povBooks),
    tv_apperance = lengths(tvSeries),
    allegiance_houses = lengths(allegiances)
  ) %>%
  select(
    -c(died, culture,
       books, born, titles, aliases, 
       povBooks, tvSeries, father, 
       mother, spouse, playedBy)
  ) %>% 
  # Take care of edge cases: some chars have multiple entries - use the unique character id to filter out duplicates
  filter(!url %in% c("https://www.anapioficeandfire.com/api/characters/271",
                  "https://www.anapioficeandfire.com/api/characters/206", 
                  "https://www.anapioficeandfire.com/api/characters/207",
                  "https://www.anapioficeandfire.com/api/characters/209",
                  "https://www.anapioficeandfire.com/api/characters/210",
                  "https://www.anapioficeandfire.com/api/characters/211",
                  "https://www.anapioficeandfire.com/api/characters/212", 
                  "https://www.anapioficeandfire.com/api/characters/213"
                  ))
tidy_got <- base_got %>% 
  select(-allegiances)
tidy_got
```

    ## # A tibble: 43 ?? 8
    ##    url           name  gender alive book_appearance??? book_pov_times tv_apperance
    ##    <chr>         <chr> <chr>  <lgl>            <int>          <int>        <int>
    ##  1 https://www.??? Edda??? Male   FALSE                5              1            2
    ##  2 https://www.??? Robe??? Male   FALSE                6              0            1
    ##  3 https://www.??? Jaim??? Male   TRUE                 2              3            5
    ##  4 https://www.??? Cate??? Female FALSE                2              3            3
    ##  5 https://www.??? Cers??? Female TRUE                 3              2            6
    ##  6 https://www.??? Daen??? Female TRUE                 1              4            6
    ##  7 https://www.??? Jora??? Male   TRUE                 5              0            6
    ##  8 https://www.??? Vise??? Male   FALSE                6              0            1
    ##  9 https://www.??? Jon ??? Male   TRUE                 1              4            6
    ## 10 https://www.??? Robb??? Male   FALSE                5              0            3
    ## # ??? with 33 more rows, and 1 more variable: allegiance_houses <int>

## Bonus: Produce untidy tibble with allegiances

Because of the fact that one character may have 0 or many allegiances,
including the allegiances column and then unnesting it makes our tibble
makes it very messy. For the purpose of retaining valuable allegiances
information, I decided to use another chart to store a version of our
non-tidy data, which includes unnested character allegiances. (I didn???t
end up using this data, but getting it into a tibble was a fun journey.)

``` r
filter_zero_allegiance <- base_got %>% 
  filter(allegiance_houses != 0) %>% 
  unnest_longer(allegiances) %>% 
  rowwise() %>% 
  mutate(allegiance_name = get_allegiance(allegiances))

zero_allegiances <- base_got %>% 
  filter(allegiance_houses == 0) %>% 
  mutate(
    allegiances = if_else(is.list(allegiances), "NA", ""),
    allegiance_name = "NA") 

with_allegiances <- zero_allegiances %>% 
  rbind(filter_zero_allegiance) 
with_allegiances
```

    ## # A tibble: 52 ?? 10
    ##    url            name  gender allegiances alive book_appearance??? book_pov_times
    ##    <chr>          <chr> <chr>  <chr>       <lgl>            <int>          <int>
    ##  1 https://www.a??? Robe??? Male   NA          FALSE                6              0
    ##  2 https://www.a??? Robb??? Male   NA          FALSE                5              0
    ##  3 https://www.a??? Joff??? Male   NA          FALSE                5              0
    ##  4 https://www.a??? Drogo Male   NA          FALSE                4              0
    ##  5 https://www.a??? Stan??? Male   NA          TRUE                 5              0
    ##  6 https://www.a??? Meli??? Female NA          TRUE                 3              1
    ##  7 https://www.a??? Varys Male   NA          TRUE                 5              0
    ##  8 https://www.a??? Shae  Female NA          FALSE                5              0
    ##  9 https://www.a??? Ygri??? Female NA          FALSE                4              0
    ## 10 https://www.a??? Gend??? Male   NA          TRUE                 4              0
    ## # ??? with 42 more rows, and 3 more variables: tv_apperance <int>,
    ## #   allegiance_houses <int>, allegiance_name <chr>

# Explore the Data

## Character death bar chart

``` r
tidy_got %>% 
  ggplot(aes(x = alive)) +
    geom_bar() + 
  labs(
    y = "Number of characters",
    title = "Character death bar chart",
    subtitle = "Source: API of Fire and Ice"
  )
```

![](GOT_files/figure-gfm/plot:%20character%20death-1.png)<!-- -->

On top, we see the general character death chart for all main GOT
characters: around 13 dead, and 31 alive. That???s a stunning 41% dead,
and they???re all main characters too! Game of Thrones is not afraid to
kill its darlings.

## Character death by gender

``` r
tidy_got %>% 
  ggplot(aes(x = alive, fill = gender)) +
  geom_bar() + 
  labs(
    y = "Number of characters",
    title = "Character death by gender",
    subtitle = "Source: API of Fire and Ice"
  )
```

![](GOT_files/figure-gfm/plot:%20gender-1.png)<!-- -->

In the plot above, we see the bar chart color-coded by gender.
Generally, there are fewer female characters, but the percentage of dead
female characters is even less than the percentage of living female
characters. There seems to be a correlation with between gender in that
a large proportion of the dead characters are male, which is interesting
and subverts some gendered tropes of helpless maidens in popular media.

## Character death by book apperances

``` r
tidy_got %>% 
  ggplot(aes(x = alive, fill = as_factor(book_appearance_times))) +
  geom_bar() + 
  labs(
    y = "Number of characters",
    fill = "Book apperances (out of 6)",
    title = "Character death by number of books in which a character appears",
    subtitle = "Source: API of Fire and Ice"
  )
```

![](GOT_files/figure-gfm/plot:%20book_appearance_times-1.png)<!-- -->

In the graph above, we see character death by number of books in which a
character appears. Dead characters come entirely from the upper register
of book appearances (2-6), which (funnily) means that new characters
that were invented in the show and not originally written by George R.
R. Martin are more likely to survive. On the other hand, characters that
are alive mostly appear for 1-5 books. This might make sense because the
characters that readers are most emotionally invested in appear the most
in the books, and because George R. R. Martin is evil and a great
writer, they also have canonically the most gut-wrenching deaths
(spoiler alert: a lot of the Stark family).

## Character death by number of book POVs

``` r
tidy_got %>% 
  ggplot(aes(x = alive, fill = as_factor(book_pov_times))) +
  geom_bar() + 
  labs(
    y = "Number of characters",
    fill = "Book points of view (out of 5)",
    title = "Character death by number of books a character has had a point of view",
    subtitle = "Source: API of Fire and Ice"
  )
```

![](GOT_files/figure-gfm/plot:%20book_pov_times-1.png)<!-- -->

Conversely, it seems that higher book POVs (number of books a character
has had a point of view) might be correlated with a higher chance of
survival. For those who didn???t survive, POV counts are predominantly 0,
with occasional 1 and 3. The higher POV counts like 3, 4, and 5 are
mostly reserved for characters who do survive. This would make sense: if
a character has a POV in a book, they are alive at that moment in time
(unless the character you???re reading a POV from is a ghost). Obviously,
there???s still the possibility that a character has a POV then gets
killed off later so they would only have fewer POV counts, as
represented by the ???FALSE??? alive column.

## Character death by TV apperances

``` r
 tidy_got %>% 
  ggplot(aes(x = alive, fill = as_factor(tv_apperance))) +
  geom_bar() + 
  labs(
    y = "Number of characters",
    fill = "TV season apperances (out of 6)",
    title = "Character death by number of seasons a character appears in the TV show",
    subtitle = "Source: API of Fire and Ice"
  )
```

![](GOT_files/figure-gfm/plot:%20tv_apperance-1.png)<!-- -->

The number of seasons in which a character appears in is the most
non-brainer factor that might contribute to a character???s survival. This
API???s data has only been updated up to season 6, which is unfortunate
and explains why none of the characters have had more than 6 TV season
appearances (for context, GOT has had 8 total seasons). But it seems
that a higher TV appearance is a higher chance of being alive for these
characters. For example, every character that made it to all 6 seasons
of the show is alive.

## Summary

Thus, it seems like there is some correlation between gender and
character death (more likely to die if male), book appearances and
character death (more likely to die if appear more in books), book POVs
and character death (more likely to die if have less POVs in books), and
TV appearances and character death (more likely to die if appear less in
TV show).

Figures are well and good, but what if we want to *predict* whether or
not a GOT character will die? For that, we turn to machine learning
modeling.

# Machine Learning Modeling

``` r
# Set seed for reproducibility
set.seed(123)

# Change type of alive column to factor for machine learning modeling
ml_got <- tidy_got %>% 
  mutate_at("alive", as.factor)

# Split data into training and testing sets
got_split <- initial_split(ml_got, prop = .75, strata = alive)
got_train <- training(got_split)
got_test <- testing(got_split)

# Create 5 folds
folds <- vfold_cv(got_train, v = 5)
```

## Logistic Regression model

``` r
# Set seed for reproducibility
set.seed(123)

# Create formula
lr_formula <- formula(alive ~ book_appearance_times + book_pov_times + tv_apperance + gender)

# Create logistic regression model
lr_mod <- logistic_reg() %>% 
  set_engine("glm") %>% 
  set_mode("classification")
  
# Bundle formula and model together into the same workflow
lr_wf <- workflow() %>% 
  add_formula(lr_formula) %>% 
  add_model(lr_mod)

# Implement 5-fold cross validation and collect metrics
lr_wf %>% 
  fit_resamples(folds) %>% 
  collect_metrics()
```

    ## # A tibble: 2 ?? 6
    ##   .metric  .estimator  mean     n std_err .config             
    ##   <chr>    <chr>      <dbl> <int>   <dbl> <chr>               
    ## 1 accuracy binary     0.838     5  0.0529 Preprocessor1_Model1
    ## 2 roc_auc  binary     1         5  0      Preprocessor1_Model1

The logistic regression model has an accuracy of 0.838 with a standard
error of 0.052 (seed set at 123).

## Logistic Regression model with tuned hyperparameters

``` r
# Set seed for reproducibility
set.seed(123)

# Create new logistic regression model with hyperparameters to tune 
lr_tune_mod <- logistic_reg(
  penalty = tune(),
  mixture = tune()
) %>% 
  set_engine("glmnet") %>% 
  set_mode("classification")

# Use grid_regular() to find 25 different tuning combinations to attempt
lr_tune_grid <- grid_regular(
  penalty(),
  mixture(),
  levels = 5
)

# Update tuned logistic regression model in workflow
lr_tune_wf <- lr_wf %>% 
  update_model(lr_tune_mod)

# Tune hyperparameters with tune_grid()
lr_tune <-
  lr_tune_wf %>%
  tune_grid(
    resamples = folds,
    grid = lr_tune_grid
  )

# Show metrics for tuned hyperparameters
lr_tune %>%
  show_best("accuracy")
```

    ## # A tibble: 5 ?? 8
    ##        penalty mixture .metric  .estimator  mean     n std_err .config          
    ##          <dbl>   <dbl> <chr>    <chr>      <dbl> <int>   <dbl> <chr>            
    ## 1 0.00316         0.25 accuracy binary     0.867     5  0.0972 Preprocessor1_Mo???
    ## 2 0.00316         0.5  accuracy binary     0.867     5  0.0972 Preprocessor1_Mo???
    ## 3 0.00316         0.75 accuracy binary     0.867     5  0.0972 Preprocessor1_Mo???
    ## 4 0.00316         1    accuracy binary     0.867     5  0.0972 Preprocessor1_Mo???
    ## 5 0.0000000001    0    accuracy binary     0.838     5  0.0914 Preprocessor1_Mo???

The best logistic regression model with tuned hyperperameters of penalty
and mixture has an accuracy of 0.866 with a standard error of 0.009
(seed set at 123).

## Random forest model with tuned hyperparameters

``` r
# Set seed for reproducibility
set.seed(123)

# Create random forest model
rf_mod <- 
  rand_forest(
    min_n = tune(),
    trees = tune()
  ) %>% 
  set_engine("ranger") %>% 
  set_mode("classification")

# Use grid_regular() to find 25 different tuning combinations to attempt
rf_grid <- grid_regular(
  min_n(),
  trees(),
  levels = 5
)

# Update boosted tree model in workflow
rf_wf <- lr_wf %>%
  update_model(rf_mod) 

# Tune hyperparameters with tune_grid()
rf_tune <-
  rf_wf %>%
  tune_grid(
    resamples = folds,
    grid = rf_grid
  )

# Show metrics for tuned hyperparameters
rf_tune %>%
  show_best("accuracy")
```

    ## # A tibble: 5 ?? 8
    ##   trees min_n .metric  .estimator  mean     n std_err .config              
    ##   <int> <int> <chr>    <chr>      <dbl> <int>   <dbl> <chr>                
    ## 1  2000     2 accuracy binary     0.905     5 0.0391  Preprocessor1_Model21
    ## 2  1500    11 accuracy binary     0.848     5 0.0456  Preprocessor1_Model17
    ## 3   500     2 accuracy binary     0.843     5 0.00583 Preprocessor1_Model06
    ## 4  1500     2 accuracy binary     0.843     5 0.00583 Preprocessor1_Model16
    ## 5  1000    11 accuracy binary     0.819     5 0.0525  Preprocessor1_Model12

The best random forest model with tuned hyperperameters has an accuracy
of 0.904 with a standard error of 0.039 (seed set at 123). This is our
best model so far, so let???s choose this one for our last fit!

## Last fit: best random forest model

``` r
# Set seed for reproducibility
set.seed(123)

# Update last fit random tree model with tuned hyperparameters min_n and trees
last_fit_mod <- 
  rand_forest(
    min_n = 2,
    trees = 2000
  ) %>% 
  set_engine("ranger") %>% 
  set_mode("classification")

# Update last fit tuned random tree model in workflow
last_fit_wf <- lr_wf %>% 
  update_model(last_fit_mod)

# Collect metrics
last_fit <- last_fit_wf %>% 
  last_fit(got_split) %>% 
  collect_metrics()
last_fit
```

    ## # A tibble: 2 ?? 4
    ##   .metric  .estimator .estimate .config             
    ##   <chr>    <chr>          <dbl> <chr>               
    ## 1 accuracy binary         0.818 Preprocessor1_Model1
    ## 2 roc_auc  binary         0.812 Preprocessor1_Model1

The random forest model with tuned hyperperameters has an accuracy of
0.818 on our held-out testing data (seed set at 123). This means that
the model we chose can correctly predict whether or not a character dies
81.8% of the time. Better than most Game of Thrones audiences. Pretty
good!
