## Introduction & Background

Food preparation holds economic, social, and personal significance.
While there are a handful of academic books and articles about the
significance of food preparation and cookbooks throughout the years, I
have yet to find a data set that gives a wide view of how cookbook
topics changed over the past century. After sharpening my skills and
recognizing the power of APIs with my past project, I decided to answer
the question myself as my second personal data analysis project.

### Data Sources

[ISBNDB](https://isbndb.com/)

I began this project in late October 2022 and took several breaks for
the holidays. The preliminary Power BI graphs were made with data
gathered on 10/31/2022 and 11/01/2022.

All other data was pulled from ISBNDB on 1/05/2023.

*Non-data sources listed after conclusion*

### Limitations

This project looked at English-language cookbooks. My personal education
and interest are in English-language cookbooks; data cleaning would have
been more complicated if I were searching for subjects in languages in
which I am unacquainted.

The initial data set excluded any ISBN with a NULL “subject” value. To
get a more comprehensive view of the data, this project includes the
unfiltered results and uses words from the title instead of a “subject”
name. Displaying both the subset with listed subjects and the larger
group of cookbooks, the limitation was minimized.

Older book data may be harder to find digitized and included in the
ISBNDB. The analyses in this project are based on relative changes and
not the exact numbers from one year to the next.

### Questions

1.  How have cookbook topics changed from 1900?

2.  What topics have the total largest number of books published since
    1900?

3.  How did these top topics change over the years and what are the
    possible reasons?

4.  How have cookbook title words changed from 1900?

## Data Collection with Python

Using ISBNDB, I pulled the information for books from 1899 and 2023 with
the search term “cookbook.” For each year, the while loop runs through
the pages of results. Once the loop reaches an exception (e.g., no more
data to pull), it goes on to the next year. The result is a list of
books listed in one Excel sheet:

``` python
import pandas as pd
import requests as req

h = {'Authorization': '48630_2f3f67a0b3aab589c6593cbdc3714d12'}
page = 1
page_size = 1000
max_page = 11
startrow = 1
year = 1899
max_year = 2023

writer = pd.ExcelWriter('./final/Books_Cookbook.xlsx', engine='xlsxwriter',engine_kwargs={'options':{'strings_to_numbers':True}})

header = True

while year < max_year:
    try:
        while page < max_page:
            responses = req.get("https://api.pro.isbndb.com/books/cookbook?year="+str(year)+"&page="+str(page)+"&pageSize="+str(page_size), headers=h)
            df = pd.json_normalize(responses.json()["books"])
            df = df[['publisher','image','synopsis','language','title_long','edition','pages','date_published','subjects','authors','title','isbn13','isbn','msrp']]
            df.to_excel(writer, sheet_name = "data", header = header,startrow=startrow)
            page += 1
            header = False
            startrow += 1000
    except Exception as e:
        print('An exception has occurred. Giving up on loop:')
        print(e)
    page = 1
    year += 1
    print(year)   

#writer.save()
##commented to prevent saving over existing files
print(df)
```

  
I repeated the process using the search term “cooking” to return even
more books for the data set:

``` python
import pandas as pd
import requests as req

h = {'Authorization': '48630_2f3f67a0b3aab589c6593cbdc3714d12'}
page = 1
page_size = 1000
max_page = 11
startrow = 1
year = 1899
max_year = 2023

writer = pd.ExcelWriter('./final/Books_Cooking.xlsx', engine='xlsxwriter',engine_kwargs={'options':{'strings_to_numbers':True}})

header = True

while year < max_year:
    try:
        while page < max_page:
            responses = req.get("https://api.pro.isbndb.com/books/cooking?year="+str(year)+"&page="+str(page)+"&pageSize="+str(page_size), headers=h)
            df = pd.json_normalize(responses.json()["books"])
            df = df[['publisher','image','synopsis','language','title_long','edition','pages','date_published','subjects','authors','title','isbn13','isbn','msrp']]
            df.to_excel(writer, sheet_name = "data", header = header,startrow=startrow)
            page += 1
            header = False
            startrow += 1000
    except Exception as e:
        print('An exception has occurred. Giving up on loop:')
        print(e)
    page = 1
    year += 1
    print(year)   

#writer.save()
##commented to prevent saving over existing files

print(df)
```

## Data Cleaning with Excel

I used Excel to clean and format the data for analysis:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Cleaning_Log.png?raw=true)

## Data Cleaning with SQL

SQL (BigQuery) was the best option for this project for the large amount
of data and the need for a join.

The data from the “cookbook” and “cooking” results were joined, omitting
any result with a NULL subject value. The brackets around the subjects
and author names were removed before isolating the subject terms and
authors. This allowed for individual analysis of the subject labels.
This joined data was saved in a new table “all_books”:

``` python
CREATE TABLE `books-367405.cookbooks.all_books` AS
  SELECT
    *,
    split(topics,",") as split_subjects,
    split(author_list,"', '") as author_names
  FROM
   (SELECT
      *,
      replace(replace(replace(subjects,"[",""),"]",""),"'","") AS topics,
      replace(replace(authors,"[",""),"]","") AS author_list
    FROM
      (SELECT
        *
      FROM
        `books-367405.cookbooks.books_cooking`
      UNION ALL
      SELECT
        *
      FROM
        `books-367405.cookbooks.books_cookbook`)

    WHERE
      subjects IS NOT NULL
    )
```

  
The following query looks daunting, but the bulk is the renaming of
categories, as the subject labels varied in case, spacing, and special
characters. The WHEN clauses under SELECT CASE normalize these labels.
Several subjects were also in lists separated with “-\>” and those
needed to be removed. Computer science ‘cookbooks’ were filtered out, as
well as subjects that were codes or numbers. The results were saved as a
list of subjects with corresponding ISBN, edition, language, and year
published information:

``` python
SELECT CASE
  #Food Categories
   WHEN lower_topics LIKE 'cooking - general & miscellaneous' OR lower_topics LIKE "%cooking & food%" OR lower_topics LIKE "%general & miscellaneous cooking%" OR lower_topics LIKE "cookery" THEN 'Cooking'
   WHEN lower_topics LIKE '%baking%' OR lower_topics LIKE "%pastry%" OR lower_topics LIKE "%dessert%" OR lower_topics LIKE "%confec%" THEN 'Baking & Pastry'
   WHEN lower_topics LIKE '%seafood%' OR lower_topics LIKE "%fish%" THEN 'Fish & Seafood'
   WHEN lower_topics LIKE '%game%' OR lower_topics LIKE "%meat%" THEN 'Meat & Game Cooking'
   WHEN lower_topics LIKE '%vegetable%' THEN 'Vegetables'
   WHEN lower_topics LIKE '%fruit%' THEN 'Fruit'
   WHEN lower_topics LIKE '%herbs%' THEN 'Herbs'
  #Dietary Specific Restrictions
   WHEN lower_topics LIKE '%vegetarian%' OR lower_topics LIKE "%vegan%" THEN 'Vegetarian & Vegan'
   WHEN lower_topics LIKE '%special%' AND lower_topics LIKE "%diets%" THEN 'Special Diets/Conditions'
   WHEN lower_topics LIKE '%quick%' THEN 'Quick & Easy'
   WHEN lower_topics LIKE '%entertain%' THEN 'Entertaining'
   WHEN lower_topics LIKE '%health%'OR lower_topics LIKE "%nutrition%" AND lower_topics NOT LIKE "%weight%" THEN 'Nutrition & Health'
   WHEN lower_topics LIKE '%weight%' AND lower_topics NOT LIKE "%diet%" THEN 'Weight Loss or Control'
   WHEN lower_topics LIKE '%low fat%' OR lower_topics LIKE "%low-fat%" THEN 'Low-Fat Diet'
   WHEN lower_topics LIKE '%low carb%' OR lower_topics LIKE "%low-carb%" THEN 'Low-Carb Diet'
   WHEN lower_topics LIKE '%low calorie%' OR lower_topics LIKE "%low-calorie%" THEN 'Low-Calorie Diet'
   WHEN lower_topics LIKE '%low cholesterol%' OR lower_topics LIKE "%low-cholesterol%" THEN 'Low-Cholesterol Diet'
   WHEN lower_topics LIKE '%low salt%' OR lower_topics LIKE "%low-salt%" or lower_topics LIKE "%salt-free%" THEN 'Low-Salt or Salt-Free Diet'
   WHEN lower_topics LIKE '%gluten-free%' THEN 'Gluten-Free Diet'
   WHEN lower_topics LIKE '%reduc%' THEN 'Reducing Diet'
   WHEN lower_topics LIKE 'diet therapy' THEN 'Diet Therapy'
   WHEN lower_topics LIKE '%diabet%' THEN 'Diabetic & Sugar Free Cooking'
   WHEN lower_topics LIKE '%jewish%' THEN 'Jewish & Kosher Cooking'

  #Geographical & Cultural Caterogires
   WHEN lower_topics LIKE '%southern%' THEN 'American-Southern Style'
   WHEN lower_topics LIKE '%southwest%' THEN 'American-Southwestern Style'
   WHEN lower_topics LIKE '%california style%' THEN 'American-California Style'
   WHEN lower_topics LIKE '%louisiana style%' THEN 'American-Louisiana Style'
   WHEN lower_topics LIKE '%midwestern style%' THEN 'American-Midwestern Style'
   WHEN lower_topics LIKE '%new england%' THEN 'American-New England Style'
   WHEN lower_topics LIKE '%pacific northwest style%' THEN 'American-Pacific Northwest Style'
   WHEN lower_topics LIKE 'french' THEN 'French Cooking'
   WHEN lower_topics LIKE '%ital%' THEN 'Italian Cooking'
   WHEN lower_topics LIKE 'chinese' THEN 'Chinese Cooking'
   WHEN lower_topics LIKE '%asian cooking%' OR lower_topics LIKE "asian" THEN 'Asian Cooking'
   WHEN lower_topics LIKE 'mexican' THEN 'Mexican Cooking'
   WHEN lower_topics LIKE 'american' OR lower_topics LIKE "american cooking" THEN 'American Cooking'

  #Seperates topics with arrow deliminated subtopics 
   WHEN topics LIKE "%->%" THEN TRIM(REGEXP_EXTRACT(topics, r".*->(.*)"))
   ELSE topics
END as Subject_Label, * 

FROM
  (SELECT TRIM(new_subjects) as topics, lower(TRIM(new_subjects)) as lower_topics, isbn, edition, language, year_published
    FROM
      `books-367405.cookbooks.all_books`,
      UNNEST(split_subjects) as new_subjects
    WHERE
      lower(subjects) NOT LIKE "%computer%" AND
      lower(subjects) NOT LIKE "%cs%" AND
      lower(language) LIKE "%en%"      
      
  )
#filter out computer science 'cookbooks' and numerical categories
WHERE
  REGEXP_CONTAINS(topics,"[a-z]") AND
  topics NOT LIKE "%[0-9]%"
  
#GROUP BY
  #Subject_Label
#HAVING
  #Instances > 10
#ORDER BY
  #Instances DESC
; 
```

  
This similar to the query above but does not omit books with NULL
subjects. This joined data was saved in a new table
“all_books_missing_subjects,” to analyze the title words:

``` python
CREATE TABLE `books-367405.cookbooks.all_books_missing_subjects` AS
  SELECT
    *,
    split(topics,",") as split_subjects,
    split(author_list,"', '") as author_names
  FROM
   (SELECT
      *,
      replace(replace(replace(subjects,"[",""),"]",""),"'","") AS topics,
      replace(replace(authors,"[",""),"]","") AS author_list
    FROM
      (SELECT
        *
      FROM
        `books-367405.cookbooks.books_cooking`
      UNION ALL
      SELECT
        *
      FROM
        `books-367405.cookbooks.books_cookbook`)
    )
```

  
The following query separates the title words, like the code for the
subject data set. The only difference is that the spaces and dashes are
removed. The results were saved as a list of title words, with
corresponding ISBN, edition, and year published:

``` python
WITH Table1 AS (
SELECT
  CASE
    WHEN REGEXP_CONTAINS(title_long," ") THEN split(title_long," ")
    WHEN REGEXP_CONTAINS(title_long,"-") THEN split(title_long,"-")
    WHEN REGEXP_CONTAINS(title_long,"_") THEN split(title_long,"_")
  END as title_topics,isbn,year_published,title_long,edition

  FROM
    (SELECT isbn, language, year_published,title_long,subjects,edition
      FROM
        `books-367405.cookbooks.all_books_missing_subjects`
      WHERE
        (lower(subjects) NOT LIKE "%computer%" AND
        lower(subjects) NOT LIKE "%cs%" AND
        lower(language) LIKE "%en%"  ) OR
        subjects IS NULL
    )
  #filter out computer science 'cookbooks' and numerical categories
)

SELECT
 isbn, edition,regexp_replace(new_topics, '[^a-zA-Z0-9]', '') as title_words,year_published
FROM
Table1,UNNEST(title_topics) as new_topics
```

## Data Analysis with Power BI and Tableau

Noticing the additional editions included in the data, I used Power BI
to explore the possibility of it skewing the data.

For the data with listed subjects, the spike is more defined, but occurs
in the 1st Edition data as well:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Visualization_Filtered.png?raw=true)  
  

In the following graph with the unfiltered (aka title word) data, the
difference is even smaller, both lines following the same increases and
decreases:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Visualization_UnFiltered.png?raw=true)  
  

The biggest difference is between the unfiltered data and the data with
listed subjects. The rises and falls, especially at key years like 2007
and 2013, vary in shape. This might be due to overall cookbook
publishing not catching up as immediately as the subject data set
suggests:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Visualization_Overview.png?raw=true) For example, there is an increase after
2007 (solid line), but the fall after 2013 (dashed line) is less
consistent compared to the listed subjects. Despite these differences,
the overall trends of rising and falling do match up, the subject data
almost acting as a trend line. For the unfiltered data, there is a big
decline in 2020, then publishing spikes again in 2021, but subject data
shows a more consistent decline. The decision to include both the
unfiltered (title word data) and the subject data is to give a more
comprehensive view.

*The rest of the visualizations in the project were made in Tableau, as
I am only now teaching myself Power BI.*

### 1. How have cookbook topics changed from 1900?

The number of books and topics increased from 1900 to 2022. There are
multiple potential causes for these changes when looking at the data:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Cookbooks%20Published%20with%20Listed%20Subjects%20by%20Year.png?raw=true)
**There is an increase in the number of cookbooks published after
1970.** Potentially attributable to a lack of digitized book data.
Another possibility is that more individuals maintained personal and
familial recipes, lessening the demand for commercial cookbooks.
Furthermore, the cookbooks people did own were more general like The Joy
of Cooking versus owning multiple cookbooks on specific subjects.
Additionally, in the 1960s, iconic individuals like Craig Claiborne and
Julia Child, as well as publications like the New York Times Cookbook,
increased the scope of home cooking (Wolf, 2006). All these factors
contributed to the rise of cookbooks in the 1970s.  
  
**Cookbook publishing increased after 2007.** This corresponds to the
beginning of the 2007-2008 financial crisis. Restaurant visits at the
beginning of the recession did not initially show a substantial decline.
But, as time passed restaurants were affected by people decreasing their
money eating out, especially at sit-down restaurants (Maynard, 2018).
The Dow Jones U.S. Restaurants & Bar Index dropped around 13% in 2008
(CBS, 2008). Individuals would then need to prepare more meals at home,
which could have led to an increase in cookbooks published.  
  
**Cookbook publishing decreased around 2013.** This corresponds to when
Grubhub went public, merged with Seamless and became a powerhouse in the
food delivery industry (Curry, 2022). The ease of delivery could have
decreased the demand for cooking meals at home.  
  
**The increase starting around 2020 and 2021** was potentially the
combined effect of economic changes and delivery availability. The
quarantine pushed people to cook their own food, like in 2007, but
benefiting from the ease of food delivery that has only increased since
2013.  

### 2. What subjects have the largest number of books published since 1900?

The top book subjects are **Cooking**, **Italian Cooking**,
**Professional**, **General**, **Quick & Easy**, **Nutrition & Health**,
**Literary Collections**, **American Cooking**, and **Vegetarian &
Vegan**:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Top%20Cookbook%20Subjects.png?raw=true)

### 3. How did these top topics change over the years and what are the possible reasons?

Looking at the top five topics over time and omitting generic subjects
like Professional and Literary Collections, the following analysis
includes **Cooking**, **Italian Cooking**, **General**, **Quick & Easy**
and **Nutrition & Health**. The line graphs show the difference in the
actual count of books and the percentage of all books published with
listed subjects:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Top%20Cookbook%20Subjects%20by%20Year%20Published.png?raw=true)  
**Italian cookbooks rose significantly in the nineties and early 2000s,
both in quantity and percentage of books published.** Italian food in
the late 1800s was viewed as “foreign” and did not receive positive hype
until the mid-to-late 1900s (McMillan, 2016). In 2016, 100,000 out of
800,000 restaurants (12.5%) in the United States served Italian food
(McMillan, 2016). The percentage of Italian cookbooks decreased around
2012 (dotted line), which corresponds to the height of diet and
reducing-focused cookbooks.  

**“Low” (low fat, low carb, etc.) and diet cookbooks rose in the
nineties and reached their peak in 2013:**

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Diet%20or%20Reducing%20Cookbooks%20by%20Year%20Published.png?raw=true)

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/_Low_%20Diet%20Cookbooks%20by%20Year%20Published.png?raw=true)
Breaking down the “Low” Diets, “Low-Fat” was the most popular, peaking
in the late 90s.

*These numbers are in the 10s, as the specific subject data for diet
cookbooks was relatively small. A larger set would be more helpful to
give a more substantial and clear-cut conclusion*

### 4. What title words are the most common since 1900?

Looking at the top topics over time and omitting generic title words
like Cookbook, Cooking, and Recipes that skew the data: **Easy**,
**Delicious**, **Healthy**, **Food**, **Diet**, **Guide**, **Complete**,
and **Quick** are the top book terms:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Top%20Cookbook%20Title%20Words%20Omitting%20General%20Terms.png?raw=true)  
  
Looking at the top title words, all of them rise and fall at similar
times, when looking at both the count and the percentage of all
published books. “Easy” is consistently at the top, rising to above 25%
in 2020. The buzzword of “Delicious” increased to surpass “Easy” in 2014
(dotted line) around the 10% point:

![](https://github.com/Maria-Catherine/Cookbook-Trends/blob/main/Visualizations/Top%20Cookbook%20Title%20Words%20by%20Year%20Published.png?raw=true)

## Conclusion & Takeaways

Economic, social, and personal changes all affect the world of food
consumption. Analyzing the changing trends in cookbooks can highlight
how all these facets of life change and evolve home cooking. Beyond
personal interest and curiosity, the data could be useful for:

**1. Identifying popular or rising trends**

**2. Identifying a void or opening in the market for cookbooks**

**4. Patterns of economics and home cooking**

**5. Patterns between social feelings toward certain groups and food**  

#### **Thank you for reading!**

##### To look closer at the visualizations: [Tableau Visualizations](https://public.tableau.com/app/profile/maria.catherine4989/viz/CookbookTrends/CookbooksPublishedwithListedSubjectsbyYear)

## References

<div style="text-indent: -40px; padding-left: 40px;">

CBS Interactive. (2008, December 31). Recession Took a Bite of
Restaurant Sales. CBS News. Retrieved January 3, 2023, from
<https://www.cbsnews.com/news/recession-took-a-bite-of-restaurant-sales/>

Curry, D. (2022, September 6). Grubhub revenue and usage statistics
(2022). Business of Apps. Retrieved January 3, 2023, from
<https://www.businessofapps.com/data/grubhub-statistics/>

Maynard, M. (2018, July 17). Three ways restaurants have changed since
the Great Recession. Forbes. Retrieved January 3, 2023, from
<https://www.forbes.com/sites/michelinemaynard/2018/07/17/here-are-three-top-ways-that-restaurants-have-changed-since-the-great-recession/?sh=26c6ae2352e7>

McMillan, T. (2016, May 4). How Italian cuisine became as American as
Apple Pie. National Geographic. Retrieved January 3, 2023, from
<https://www.nationalgeographic.com/culture/article/how-italian-cuisine-became-as-american-as-apple-pie>

Wolf, B. (2006, May 14). The Evolution of Cookbooks \[Radio broadcast
transcript\]. In Weekend Edition Sunday. NPR. Retrieved January 3, 2023,
<https://www.npr.org/templates/story/story.php?storyId=5403698.>

</div>
