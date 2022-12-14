# 🍜 Case Study Marketing Analytics

## Data Exploration

### Join Column Analysis Example 1

1. Perform an anti join to check which column values exist in dvd_rentals.rental but not in dvd_rentals.inventory

````sql
-- how many foreign keys only exist in the left table and not in the right?
SELECT
  COUNT(DISTINCT rental.inventory_id)
FROM dvd_rentals.rental
WHERE NOT EXISTS (
  SELECT inventory_id
  FROM dvd_rentals.inventory
  WHERE rental.inventory_id = inventory.inventory_id
);
````
![image](https://user-images.githubusercontent.com/104872221/184936110-c575c65a-26a0-4af6-8237-7cff499de9ba.png)


2. Now check the right side table using the same process: dvd_rentals.inventory

````sql
-- how many foreign keys only exist in the right table and not in the left?
-- note the table reference changes
SELECT
  COUNT(DISTINCT inventory.inventory_id)
FROM dvd_rentals.inventory
WHERE NOT EXISTS (
  SELECT inventory_id
  FROM dvd_rentals.rental
  WHERE rental.inventory_id = inventory.inventory_id
);
````

![image](https://user-images.githubusercontent.com/104872221/184937107-b1ad452c-6f06-4a9a-867f-05a439a514b6.png)

3. There seems to be a single value which is not showing up - let’s investigate which film it is:

````sql
SELECT *
FROM dvd_rentals.inventory
WHERE NOT EXISTS (
  SELECT inventory_id
  FROM dvd_rentals.rental
  WHERE rental.inventory_id = inventory.inventory_id
);
````

![image](https://user-images.githubusercontent.com/104872221/184939888-0ebfdeef-96de-4b00-85c9-6886d07ef9b5.png)


Conclusion: although there is a single inventory_id record which is missing from the dvd_rentals.rental table - there might be no issues with this discrepancy as it seems that some inventory might just never be rented out to customers at the retail rental stores.

4. Finally - let’s confirm that both left and inner joins do not differ at all when we look at the resulting row counts from the joint tables:

````sql
DROP TABLE IF EXISTS left_rental_join;
CREATE TEMP TABLE left_rental_join AS
SELECT
  rental.customer_id,
  rental.inventory_id,
  inventory.film_id
FROM dvd_rentals.rental
LEFT JOIN dvd_rentals.inventory
  ON rental.inventory_id = inventory.inventory_id;

DROP TABLE IF EXISTS inner_rental_join;
CREATE TEMP TABLE inner_rental_join AS
SELECT
  rental.customer_id,
  rental.inventory_id,
  inventory.film_id
FROM dvd_rentals.rental
INNER JOIN dvd_rentals.inventory
  ON rental.inventory_id = inventory.inventory_id;

-- Output SQL
(
  SELECT
    'left join' AS join_type,
    COUNT(*) AS record_count,
    COUNT(DISTINCT inventory_id) AS unique_key_values
  FROM left_rental_join
)
UNION
(
  SELECT
    'inner join' AS join_type,
    COUNT(*) AS record_count,
    COUNT(DISTINCT inventory_id) AS unique_key_values
  FROM inner_rental_join
);
````

![image](https://user-images.githubusercontent.com/104872221/184940310-b0761888-90c0-49c5-a191-d53028063886.png)

We perform this same analysis for all of our tables within our core tables and concluded that the distribution for each of the join keys are as expected and are similar to what we see for these first 2 tables.


### 4. Join Column Analysis Example 2

````SQL
WITH film_actor_counts AS (
SELECT
  actor_id,
  COUNT(DISTINCT film_id) AS total_films
FROM dvd_rentals.film_actor
GROUP BY 1
)

SELECT 
  total_films,
  COUNT(actor_id) AS number_of_actors
FROM film_actor_counts
GROUP BY total_films
ORDER BY total_films DESC;
````

![image](https://user-images.githubusercontent.com/104872221/184943191-d7d9f527-358e-4bcb-9d85-0b2231cc0b79.png)

Confirm there are multiple actors per film.

````sql
WITH film_actor_counts AS (
  SELECT
    film_id,
    COUNT(DISTINCT actor_id) AS actor_count
  FROM dvd_rentals.film_actor
  GROUP BY film_id
)
SELECT
  actor_count,
  COUNT(*) AS total_films
FROM film_actor_counts
GROUP BY actor_count
ORDER BY actor_count DESC;
````
Well - what do you know? Some films only consist of a single actor! Good thing we checked after all!



![image](https://user-images.githubusercontent.com/104872221/184961108-5b20073b-736e-4cfa-9829-ecf2efca7b8c.png)

In conclusion - we can see that there is indeed a many to many relationship of the film_id and the actor_id columns within the dvd_rentals.film_actor table so we must take extreme care when we are joining these 2 tables as part of our analysis in the next section of this project!


## Analysis

Now that we’ve identified the key columns and highlighted some things we need to keep in mind when performing some table joins for our data analysis - we need to formalise our plan of attack.

Before we dive into the actual SQL implementation of the final solution, let’s list out all of the steps we will take without any code so we can keep a track of our thought process as we go through the following SQL solution.

At a high level - we will tackle the category insights first before turning our attention to the actor level insights and recommendations.

### Solution Plan

#### Category Insights

1. Create a base dataset and join all relevant tables
      * *complete_joint_dataset*
2. Calculate customer rental counts for each category
      * *category_counts*
3. Aggregate all customer total films watched
      * *total_counts*
4. Identify the top 2 categories for each customer
      * *top_categories*
5. Calculate each category’s aggregated average rental count
      * *average_category_count*
6. Calculate the percentile metric for each customer’s top category film count
      * *top_category_percentile*
7. Generate our first top category insights table using all previously generated tables
      * *top_category_insights*
8. Generate the 2nd category insights
      * *second_category_insights*

#### Category Recommendations
1. Generate a summarised film count table with the category included, we will use this table to rank the films by popularity
      * *film_counts*
2. Create a previously watched films for the top 2 categories to exclude for each customer
      * *category_film_exclusions*
3. Finally perform an anti join from the relevant category films on the exclusions and use window functions to keep the top 3 from each category by popularity - be sure to split out the recommendations by category ranking
      * *category_recommendations*

#### Actor Insights
1. Create a new base dataset which has a focus on the actor instead of category
      * actor_joint_table
2. Identify the top actor and their respective rental count for each customer based off the ranked rental counts
      * top_actor_counts
#### Actor Recommendations
1. Generate total actor rental counts to use for film popularity ranking in later steps
      * actor_film_counts
2. Create an updated film exclusions table which includes the previously watched films like we had for the category recommendations - but this time we need to also add in the films which were previously recommended
      * actor_film_exclusions
3. Apply the same ANTI JOIN technique and use a window function to identify the 3 valid film recommendations for our customers
      * actor_recommendations

#### Category Insights

We first created a complete_joint_dataset which joins multiple tables together after analysing the relationships between each table to confirm if there was a one-to-many, many-to-one or a many-to-many relationship for each of the join columns.

We also included the rental_date column to help us split ties for rankings which had the same count of rentals at a customer level - this helps us prioritise film categories which were more recently viewed.

````sql
DROP TABLE IF EXISTS complete_joint_dataset;
CREATE TEMP TABLE complete_joint_dataset AS
SELECT
  rental.customer_id,
  rental.rental_date,
  inventory.film_id,
  film.title,
  category.name AS category_name
FROM dvd_rentals.rental
INNER JOIN dvd_rentals.inventory
ON rental.inventory_id = inventory.inventory_id
INNER JOIN dvd_rentals.film
ON inventory.film_id = film.film_id
INNER JOIN dvd_rentals.film_category
ON film.film_id = film_category.film_id
INNER JOIN dvd_rentals.category
ON film_category.category_id = category.category_id;

SELECT * FROM complete_joint_dataset
LIMIT 10;

````
![image](https://user-images.githubusercontent.com/104872221/185016886-96f76539-65be-455a-9545-ad087aa3d02e.png)


#### Category Counts
We then created a follow-up table which uses the complete_joint_dataset to aggregate our data and generate a rental_count and the latest rental_date for our ranking purposes downstream.

````sql
DROP TABLE IF EXISTS category_counts;
SELECT
  customer_id,
  category_name,
  COUNT(*) AS rental_count,
  MAX(rental_date) AS latest_rental_date
FROM complete_joint_dataset
GROUP BY 1,2
````

````SQL
SELECT
  *
FROM category_counts
WHERE customer_id = 1
ORDER BY 
  rental_count DESC,
  latest_rental_date DESC;
````

![image](https://user-images.githubusercontent.com/104872221/185017340-80423167-0389-4d5b-9586-ae1eb2a3d587.png)


#### Total Counts
We will then use this category_counts table to generate our total_counts table.

````sql
DROP TABLE IF EXISTS total_counts;
CREATE TEMP TABLE total_counts AS 
SELECT
  customer_id,
  SUM(rental_count) AS total_counts
FROM category_counts
GROUP BY customer_id;


SELECT 
  *
FROM total_counts
LIMIT 5;
````

![image](https://user-images.githubusercontent.com/104872221/185271062-dba8056e-4892-4f50-841c-260b7d7d8ee9.png)

#### Top Categories
We can also use a simple DENSE_RANK window function to generate a ranking of categories for each customer.

We will also split arbitrary ties by preferencing the category which had the most recent latest_rental_date value we generated in the category_counts for this exact purpose. To further prevent any ties - we will also sort the category_name in alphabetical (ascending) order just in case!

````SQL

DROP TABLE IF EXISTS top_categories;
CREATE TEMP TABLE top_categories AS 
WITH ranked_cte AS (
  SELECT 
    customer_id,
    category_name,
    rental_count,
    DENSE_RANK() OVER (
      PARTITION BY customer_id
      ORDER BY 
        rental_count DESC,
        latest_rental_date DESC,
        category_name
        ) AS category_rank
  FROM category_counts
)
SELECT * FROM ranked_cte
WHERE category_rank <= 2;


  
SELECT 
  *
FROM top_categories
LIMIT 5;

````

![image](https://user-images.githubusercontent.com/104872221/185273536-417fedcc-e9de-4cd3-bf1a-7c1f4b5a240e.png)

#### Average Category Count

Next we will need to use the category_counts table to generate the average aggregated rental count for each category rounded down to the nearest integer using the FLOOR function.


````SQL
DROP TABLE IF EXISTS average_category_count;
CREATE TEMP TABLE average_category_count AS 
SELECT
  category_name,
  FLOOR(AVG(rental_count)) AS avg_rental_count
FROM category_counts
GROUP BY category_name;

````



<details>
<summary>
Click here to see sample rows from average_category_count
</summary>

  
````SQL
SELECT *
FROM average_category_count
ORDER BY
category_average DESC,
category_name;
````
![image](https://user-images.githubusercontent.com/104872221/185275110-2630a00b-8eb5-49c5-96ef-880abc3a24ea.png)
</details>


#### Top Category Percentile
Now we need to compare each customer’s top category rental_count to all other DVD Rental Co customers - we do this using a combination of a LEFT JOIN and a PERCENT_RANK window function ordered by descending rental count to show the required top N% customer insight.

We will also use a CASE WHEN to replace a 0 ranking value to 1 as it doesn’t make sense for the customer to be in the Top 0%!

````sql
DROP TABLE IF EXISTS top_category_percentile;
CREATE TEMP TABLE top_category_percentile AS 
WITH calculated_cte AS (
SELECT
  top_categories.customer_id,
  top_categories.category_name AS top_category_name,
  top_categories.rental_count,
  category_counts.category_name,
  top_categories.category_rank,
  PERCENT_RANK() OVER (
    PARTITION BY category_counts.category_name
    ORDER BY category_counts.rental_count DESC) AS raw_percentile_value
FROM category_counts
LEFT JOIN top_categories
ON top_categories.customer_id = category_counts.customer_id
)
SELECT
  customer_id,
  category_name,
  rental_count,
  category_rank,
  CASE
    WHEN ROUND(100 * raw_percentile_value) = 0 THEN 1
    ELSE ROUND(100 * raw_percentile_value)
  END AS percentile
FROM calculated_cte
WHERE
  category_rank = 1
  AND top_category_name = category_name;
````

<details>
<summary>
Click here to see sample rows from average_category_count
</summary>
  
  
  
````sql
 SELECT *
FROM top_category_percentile
LIMIT 10;
````
 
![image](https://user-images.githubusercontent.com/104872221/185435721-bfedd85f-07e0-413a-a4dc-4c8d0aafc941.png)

</details>


#### 1st Category Insights

Let’s now compile all of our previous temporary tables into a single category_insights table with what we have so far - we will use our most recently generated top_category_percentile table as the base and LEFT JOIN our average table to generate an average_comparison column.

````sql
DROP TABLE IF EXISTS first_category_insights;
CREATE TEMP TABLE first_category_insights AS
SELECT
  base.customer_id,
  base.category_name,
  base.rental_count,
  base.rental_count - average.category_average AS average_comparison,
  base.percentile
FROM top_category_percentile AS base
LEFT JOIN average_category_count AS average
  ON base.category_name = average.category_name;
````
<details>
<summary>
Click here to see sample rows from first_category_insights
</summary>

````sql
SELECT *
FROM first_category_insights
LIMIT 10;
````
  
![image](https://user-images.githubusercontent.com/104872221/185525088-490f6ce1-196d-4fdb-83c1-f466d6e1ca08.png)


</details>

  
#### 2nd Category Insights
Our second ranked category insight is pretty simple as we only need our top_categories table and the total_counts table to process our insights.
The only thing to note here is that we’ll need to cast one of our fraction components of our total_percentage column to avoid the dreaded integer floor division!

````sql
DROP TABLE IF EXISTS second_category_insights;
CREATE TEMP TABLE second_category_insights AS
SELECT
  top_categories.customer_id,
  top_categories.category_name,
  top_categories.rental_count,
  -- need to cast as NUMERIC to avoid INTEGER floor division!
  ROUND(
    100 * top_categories.rental_count::NUMERIC / total_counts.total_count
  ) AS total_percentage
FROM top_categories
LEFT JOIN total_counts
  ON top_categories.customer_id = total_counts.customer_id
WHERE category_rank = 2;
````

<details>
<summary>
Click here to see sample rows from second_category_insights
</summary>

 ````sql
  SELECT *
FROM second_category_insights
LIMIT 10;
````
![image](https://user-images.githubusercontent.com/104872221/185528270-daa3a5e0-37f7-4d98-9bb3-4ae7835645f2.png)
  
</details>


#### Category Recommendations
#### Film Counts
We wil first generate another total rental count aggregation from our base table complete_joint_dataset - however this time we will use the film_id and title instead of the category - we still need to keep the category_name in our aggregation - so we will need to use a window function instead of a group by to perform this step.

The DISTINCT is really important for this query - if we were to omit it we would end up with duplicates in our table, which is definitely not what we want!

````sql
DROP TABLE IF EXISTS film_counts;
CREATE TEMP TABLE film_counts AS
SELECT DISTINCT
  film_id,
  title,
  category_name,
  COUNT(*) OVER (
    PARTITION BY film_id
  ) AS rental_count
FROM complete_joint_dataset;
````

<details>
<summary>
Click here to see sample rows from film_counts
</summary>

````sql
  SELECT *
FROM film_counts
ORDER BY rental_count DESC
LIMIT 10;
  
````

![image](https://user-images.githubusercontent.com/104872221/185724300-b4617aa1-3f5e-4c28-bad6-98258437ae97.png)


</details>




#### Category Film Exclusions
For the next step in our recommendation analysis - we will need to generate a table with all of our customer’s previously watched films so we don’t recommend them something which they’ve already seen before.

We will use the complete_joint_dataset base table to get this information - it is pretty straightforward, the only thing to keep in mind is how we will perform an ANTI JOIN with our previous film_counts table so we need to also keep the same film_id column to use as our join column.

We also want to make sure that the DISTINCT is also applied for this table too - it is not as important as in our last step, but it would be best practice to apply it here also!

Note: we could also keep the title and category_name columns but they are redundant and we want to minimise the amount of data we need to use!

````sql
DROP TABLE IF EXISTS category_film_exclusions;
CREATE TEMP TABLE category_film_exclusions AS
SELECT DISTINCT
  customer_id,
  film_id
FROM complete_joint_dataset;
````

<details>
<summary>
Click here to see sample rows from category_film_exclusions
</summary>


![image](https://user-images.githubusercontent.com/104872221/185724688-69dde47d-3f98-45ac-96f4-04b8a79772ca.png)
  
  
</details>


#### Final Category Recommendations
Finally we have arrived at the final point of our category recommendations analysis where we need to perform an ANTI JOIN on our category_film_exclusions table using a WHERE NOT EXISTS SQL implementation for our top 2 categories found in the top_categories table we generated a few steps prior.

After this exclusion - we will then perform a window function to select the top 3 films for each of the top 2 categories per customer. To avoid random ties - we will sort by the title alphabetically in case the rental_count values are equal in the ORDER BY clause for our window function.

We also need to keep our category_rank column in our final output so we can easily identify our recommendations for each customer’s preferred categories.

This ANTI JOIN is likely to be the most complex and challenging piece to understand in this entire analysis - please do not go past this point until you understand what is going on!

````sql
DROP TABLE IF EXISTS category_recommendations;
CREATE TEMP TABLE category_recommendations AS
WITH ranked_films_cte AS (
  SELECT
    top_categories.customer_id,
    top_categories.category_name,
    top_categories.category_rank,
    -- why do we keep this `film_id` column you might ask?
    -- you will find out later on during the actor level recommendations!
    film_counts.film_id,
    film_counts.title,
    film_counts.rental_count,
    DENSE_RANK() OVER (
      PARTITION BY
        top_categories.customer_id,
        top_categories.category_rank
      ORDER BY
        film_counts.rental_count DESC,
        film_counts.title
    ) AS reco_rank
  FROM top_categories
  INNER JOIN film_counts
    ON top_categories.category_name = film_counts.category_name
  -- This is a tricky anti-join where we need to "join" on 2 different tables!
  WHERE NOT EXISTS (
    SELECT 1
    FROM category_film_exclusions
    WHERE
      category_film_exclusions.customer_id = top_categories.customer_id AND
      category_film_exclusions.film_id = film_counts.film_id
  )
)
SELECT * FROM ranked_films_cte
WHERE reco_rank <= 3;
````


<details>
<summary>
Click here to see sample rows from category_recommendations
</summary>




````sql
SELECT *
FROM category_recommendations
WHERE customer_id = 1
ORDER BY category_rank, reco_rank;
```` 
  
![image](https://user-images.githubusercontent.com/104872221/185752498-3249fa31-43cb-4bb2-811f-cb9501fc8f49.png)
  
  
</details>  
  
  
#### Actor Insights
#### Actor Joint Table

For this entire analysis on actors - we will need to create a new base table as we will need to introduce the dvd_rentals.film_actor and dvd_rentals.actor tables to extract all the required data points we need for the final output.

We should also check that the combination of rows in our final table is expected because we should see many more rows than previously used in our categories insights as there is a many-to-many relationship between film_id and actor_id as we alluded to earlier in our data exploration section of this case study.

We also included the rental_date column in this table so we can use it in case there are any ties - just like our previous analysis for the top categories piece.

````sql
-- Actor Insights and Recommendations
DROP TABLE IF EXISTS actor_joint_dataset;
CREATE TEMP TABLE actor_joint_dataset AS
SELECT
  rental.customer_id,
  rental.rental_id,
  rental.rental_date,
  film.film_id,
  film.title,
  actor.actor_id,
  actor.first_name,
  actor.last_name
FROM dvd_rentals.rental
INNER JOIN dvd_rentals.inventory
  ON rental.inventory_id = inventory.inventory_id
INNER JOIN dvd_rentals.film
  ON inventory.film_id = film.film_id
-- different to our previous base table as we know use actor tables
INNER JOIN dvd_rentals.film_actor
  ON film.film_id = film_actor.film_id
INNER JOIN dvd_rentals.actor
  ON film_actor.actor_id = actor.actor_id;
````

<details>
<summary>
Click here to see sample rows and counts from actor_joint_dataset
</summary>


````sql
-- show the counts and count distinct of a few important columns
SELECT
  COUNT(*) AS total_row_count,
  COUNT(DISTINCT rental_id) AS unique_rental_id,
  COUNT(DISTINCT film_id) AS unique_film_id,
  COUNT(DISTINCT actor_id) AS unique_actor_id,
  COUNT(DISTINCT customer_id) AS unique_customer_id
FROM actor_joint_dataset;
````
  

  
![image](https://user-images.githubusercontent.com/104872221/185759245-8b3d6194-83e2-4ffe-8f56-92d616e63e50.png)
  
 
````sql
-- show the first 10 rows
SELECT *
FROM actor_joint_dataset
LIMIT 10;
````
  
![image](https://user-images.githubusercontent.com/104872221/185759304-5ed6cd60-edfe-4f66-b355-fe255aff44cb.png)

  

</details>  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
