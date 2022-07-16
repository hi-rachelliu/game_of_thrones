# Game of Thrones Character Death Visualization and Prediction 

## 🐉 Overview
Game of Thrones is famous, or perhaps infamous, for its high number of character deaths. Using data from an [API of Ice and Fire](https://anapioficeandfire.com/) on all GOT main characters, let's visualize and predict whether or not a character might die!

## ⏰ Relevant Files 
- GOT.Rmd: an Rmd file that contains all solutions to the second part of hw08, creating new data from the Game of Thrones API. Data is obtained from the an API of Ice and Fire. No authentication is required to access this API. Note: because of the fact that this API is missing some data (character TV appearances in season 7 and 8 are not recorded, for example), some inferences and conclusions may be inaccurate.
- GOT.md: an md file that contains all code and results from GOT.Rmd
- GOT_files/figure-gfm/: a folder that contains all figures generated from gapminder.Rmd

📦 Necessary Packages

Required packages for GOT.Rmd include:

```r
library(tidyverse)
library(httr)
library(jsonlite)
library(stringr)
library(rvest)
library(tidymodels)
```

👾 Execution
Make sure that you have already installed the following packages before running GOT.Rmd:
```r
library("glmnet")
library("ranger")
```
If not, please use the following to download them: 
```r
install.packages("glmnet")
install.packages("ranger")
```