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






