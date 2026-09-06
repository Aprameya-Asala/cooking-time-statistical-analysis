## Introduction
As a college student, I am often faced with the question, should I cook at home? With how busy I am, will whatever I have the time to cook even taste good? This is what I will answer in the first part of my project, followed by me creating a predictive model on the calorie count of a recipe using its fat, sugar, and other macros. But first an introduction to the dataset I will be using. I will be using the Recipes and Ratings dataset, containing information regarding recipes from food.com originally scraped by Bodhisattwa Prasad Majumder, Shuyang Li, Jianmo Ni, and Julian McAuley. This dataset contains various information regarding recipes, with the important information for this project described below:
| Field       | Description                                                                                                                                                                                       | Type   |
|:------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-------|
| `id`        | Recipe ID                                                                                                                                                                                         | int    |
| `minutes`   | Minutes to prepare recipe                                                                                                                                                                         | int    |
| `submitted` | Date recipe was submitted                                                                                                                                                                         | str    |
| `nutrition` | Nutrition information in the form [calories (#), total fat (PDV), sugar (PDV), sodium (PDV), protein (PDV), saturated fat (PDV), carbohydrates (PDV)]; PDV stands for "percentage of daily value" | str    |
| `rating`    | User-submitted rating for the recipe (1-5 stars)                                                                                                                                                  | int    |
| `description` | User-provided description of the recipe | str |

The dataset I will be working with has 83782 rows. Keep reading to find out how I figure out if cooking is worth the time commitment, especially to those with busy lives (like college students).
## Data Cleaning and Exploratory Data Analysis
Cleaning this dataset required quite a bit of work. First I started by merging the raw recipes csv and the interactions csv, to compile a singular dataframe to work with and adding a column of average ratings. While doing this, I set all ratings of value 0 to nan, as rating cannot have the value of 0, and if it were it would affect the mean calculations. Then I created a new column called `under_hour`, containing boolean values of whether the recipe's minutes value was greater than or under an hour. I used an hour as the threshold, because as a college student I felt that it was a good standard of time to measure whether a recipe took a long time to make or was more reasonable. Then I created `minutes_log` to help plot the distribution of minutes, as it has a few extreme values that I did not want to remove. Then I turned `submitted` into a date time object to make it easier to interact with. Then I fixed the nutrition, tags, steps, and ingredients columns' issue, which was that it was storing lists as strings. I fixed this by iterating through all the items in the columns and removing the quotes around the brackets which caused the list to register as `str` type. Then I broke each of the components of the `nutrition` column into its own columns. Then I created a column of booleans called `is_baked_good` by going through all the steps of each ingredieng and searching for the string 'bake'. Then I added a column of just the years form the `submitted` column, and a column of booleans for whether or not rating was missing for that recipe as `rating_missing`. Finally, I added a feature called `recent_submission` for when the `year` is greater than or equal to 2015. Lets take a look at some relevant columns of the cleaned up dataframe: 
| name                                 |   minutes | under_hour   |   rating |   calories |   total_fat_pdv | is_baked_good   |
|:-------------------------------------|----------:|:-------------|---------:|-----------:|----------------:|:----------------|
| 1 brownies in the world    best ever |        40 | True         |        4 |      138.4 |              10 | True            |
| 1 in canada chocolate chip cookies   |        45 | True         |        5 |      595.1 |              46 | True            |
| 412 broccoli casserole               |        40 | True         |        5 |      194.8 |              20 | True            |
| millionaire pound cake               |       120 | False        |        5 |      878.3 |              63 | True            |
| 2000 meatloaf                        |        90 | False        |        5 |      267   |              30 | False           |

Looks much better, and easier to acess. Now lets observe some of the univariate and bivariate distributions in our dataset:
<iframe
  src="assets/rating-distribution.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>
Here is a distribution of the average ratings for each recipe in the dataset portrayed using a density histogram. There is a visible left skew in the rating distribution, hinting at a pattern in how the food here was rated. It is important to understand this distribution as this is the base distribution before I start transforming it to extract insights, and observing this can help us check if there is anything wrong with the data and keep us grounded while performing analysis.
<iframe
  src="assets/cooktime_logged.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>
Above is the distribution of the logged cook time in minutes, as the original cook time had a few outliers, this is much better to inspect visually. As we can see, there are no issues with this data, and the logged minutes are approximately normally distributed. This should be kept in mind, though we do not use logged minutes in our analysis, that there are not any unforeseen issues with the data.
<iframe
  src="assets/rating-distribution-hourly.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>
And here is what the differences between our under and over hour in cook time's rating distributions look like. As observed by this density histogram, the differences seem quite small, but visible in some ratings. It will be interesting to see if there is any significance in this difference in the later steps.
<iframe
  src="assets/rating-distribution-baked.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>
And here we have a comparison on the rating distribution of baked vs unbaked goods. This density histogram shows us whether baked goods were treated differently by those who submitted reviews than unbaked goods, and at least visually there do appear to be some differences in the distributions, particularly in the rating 5 column where we can see that a greater probability of unbaked goods had ratings this high than baked goods.

Finally, lets take a look at the actual numbers for the ratings of over an under an hour, as that will be the main focus of this part of the project. Below, we can see that there is a very minor difference between the mean ratings of each category. Then the question becomes, does this difference hold any statistical significance? If so, does it matter? These are the next steps, which we will tackle after a quickly addressing the missing data in this dataset.
| under_hour   |   rating |
|:-------------|---------:|
| False        |  4.61345 |
| True         |  4.62931 |
## Assessment of Missingness
Starting off this section, there are only three columns of the dataset that have missing values, with them being `name`, `description`, and `rating`. With these columns, I do believe that one of these is MNAR, as in it is missing because of its value, namely `name` (haha get it). This is because there is only one name missing in the entire dataset, which makes me wonder how only one was missing if there was an issue. I believe that this name was missing due to its own value, and some types of columns we could add to turn this from MNAR to MAR, could be the language of the name, and the length of the name, both of which could be reasons as to why the name was not saved.

Now lets check if the `rating` column is MAR on the year the recipe was published. Performing a permutation test on the columns by shuffling `recent_submission`. The null hypothesis is that there is no difference in the missingness of `rating` whether or not it was a recent submission, with the alternative hypothesis being that there is a difference in the missingness of `rating` whether or not it was a recent submission at the 0.05 significance level. Here we can see that the results are statistically significant with p-value being essentially zero, meaning that we have sufficient evidence to reject the null hypothesis, that `recent_submission` does not effect `rating`. Below is the graph of the permutation test of the test statistic which was the absolute difference between the proportion of missing ratings.
<iframe
  src="assets/permutation-rating-year.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

Then lets look at whether `rating` is MAR on the length of names in the `name` column. The null hypothesis here is that there is no difference in the mean length of names of recipes that are missing ratings and those that have ratings, with the alternate hypothesis that there is a difference in the mean length of names of recipes that are missing ratings and those that have ratings at the 0.05 significance level. First we drop the one missing row in the `name` column before continuing. The test statistic here is the absolute difference between the mean length of names, and we are shuffling the `rating_missing`. Here the results were not statistically significant, with a p-value of 0.684, leading us to fail to reject the null hypothesis. There is insufficient evidence to reject the claim that there is no difference in the mean length of names of recipes that are missing ratings and those that have ratings.
## Hypothesis Testing
## Framing a Prediction Problem
## Baseline Model
## Final Model
## Fairness Analysis
