# Netflix-SQL-Project
# Netflix Movies and TV Shows Data Analysis using SQL

![](https://github.com/ashishkushbi/Netflix-SQL-Project/blob/main/netflix.png)

## Overview
This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL. The goal is to extract valuable insights and answer various business questions based on the dataset. The following README provides a detailed account of the project's objectives, business problems, solutions, findings, and conclusions.

## Objectives

- Analyze the distribution of content types (movies vs TV shows).
- Identify the most common ratings for movies and TV shows.
- List and analyze content based on release years, countries, and durations.
- Explore and categorize content based on specific criteria and keywords.

## Techniques used in this analysis.

- Splitting into parts and converting into numeric or int for analysis.
- String to array to convert the string value into arrays.
- Unnest convert the array into rows.
- CTE functions to  better readability by dividing the whole analysis into steps and also cte's are recursive to follow the hierarchical steps
- Sub query is similar to cte for readability but at time not reusable elsewhere.
- To-DATE function to convert the text into the date in their existing format
  
## Dataset

Origine Source -The data for this project is sourced from the Kaggle dataset:

**Dataset Link:** [Movies Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)


## Database Creation.
- Tool used for analysis **PostgreSQL**.
```
DROP TABLE IF EXISTS netflix;
CREATE TABLE netflix
(
    show_id      VARCHAR(5),
    type         VARCHAR(10),
    title        VARCHAR(250),
    director     VARCHAR(550),
    casts        VARCHAR(1050),
    country      VARCHAR(550),
    date_added   VARCHAR(55),
    release_year INT,
    rating       VARCHAR(15),
    duration     VARCHAR(15),
    listed_in    VARCHAR(250),
    description  VARCHAR(550)
);
```
## CSV file uploaded to database.
- [Download CSV file - Github ](https://github.com/ashishkushbi/Netflix-SQL-Project.git)


## Business Problems and Answers

-- 1. Count the Number of Movies vs TV Shows
```
SELECT 
    type,
    COUNT(*)
FROM netflix
GROUP BY 1;
```
```
select count(*) as total_content, type from netflix
group by 2;

```

-- 2. Find the Most Common Rating for Movies and TV Shows


-- my solution
```
select * from 
       (
		select count(rating) as total_rating, rating, type,
		dense_rank()over(partition by type order by count(rating)desc) as ranking
		from netflix
		group by 2, 3
       )
where ranking = 1;
```
-- zero analyst solution 

```
WITH RatingCounts AS (
    SELECT 
        type,
        rating,
        COUNT(*) AS rating_count
    FROM netflix
    GROUP BY type, rating
),
RankedRatings AS (
    SELECT 
        type,
        rating,
        rating_count,
        RANK() OVER (PARTITION BY type ORDER BY rating_count DESC) AS rank
    FROM RatingCounts
)
SELECT 
    type,
    rating AS most_frequent_rating
FROM RankedRatings
WHERE rank = 1;

```

-- 3. List All Movies Released in a Specific Year (e.g., 2020)
```
SELECT title, release_year FROM netflix
WHERE release_year = '2020';

```
-- 4. Find the Top 5 Countries with the Most Content on Netflix

```
select trim(unnest(string_to_array(country, ','))) as new_country, 
       count(*) as total_content 
	   from netflix
	   group by 1
	   order by count(*) desc
       limit 5;

```
-- 5. Identify the Longest Movie

```
select title as movie_name, 
       (split_part(duration,' ',1)::INT) as run_time 
	   from netflix
where type = 'Movie' and duration is not null
order by run_time desc;

```
-- 6. Find content added in the last 5 years
```
select * from netflix
where (TO_DATE(date_added,'Month DD, YYYY')) >= CURRENT_DATE - INTERVAL '5 YEARS'

```
-- 7. Find all the movies/TV shows by director 'Rajiv Chilaka'!
```
SELECT show_id, 
       title as Content_Name,
	   type as Category,
       director as Director_Name, 
	   release_year, 
	   duration 
	   FROM netflix
       Where director LIKE '%Rajiv Chilaka'
	   order by release_year desc
```

-- 8. List all TV shows with more than 5 seasons.
```
select * from netflix

Select title as TV_Show, 
       duration as Run_time
      from netflix
      where type = 'TV Show' and (split_part(duration, ' ',1)::INT) > 5 
      order by  (split_part(duration, ' ',1)::INT) desc;
```

-- 9. Count the number of content items in each genre

- without cte
```
select * from netflix;

select trim(unnest(string_to_array(listed_in,','))) as genre, 
       count(*) as Total_content
	   from netflix
       group by 1
	   order by Total_content desc;
	   
```
- with cte
```
with ct1 as ( 
                select trim(unnest(string_to_array(listed_in, ',' ))) as genre,
				       count(*) as total_content
				       from netflix
					   group by genre
				)
				select genre, total_content from ct1
				order by total_content desc;

```
-- 10. Top 10 years of content release in India on netflix. 
```
select count(*)as total_release, release_year from netflix
where country = 'India'
group by release_year
order by 1 desc;
```
-- 11.Find each year and the percentage numbers of content release in India on netflix. 

``` select*from netflix

                select release_year, 
				       country,
				       count(show_id) as total_release,
				       round((count(show_id)::numeric / (select count(show_id)
                                                                        from netflix where country ='India')::numeric *100),2)
                                                                        as release_pnt
					   from netflix
					   where country = 'India'
					   group by release_year ,country
					   order by release_pnt desc
                       limit 5
```

-- 12. List all movies that are documentaries
```
SELECT * FROM netflix
WHERE listed_in LIKE '%Documentaries%'
```


-- 13. Find all content without a director
```
Select * from netflix
where director is null
```


-- 14. Find in how many movies actor 'Salman Khan' appeared in last 10 years!
```
select Count(*)as total_content from netflix
where casts like '%Salman Khan%' and to_date(date_added, 'Month,DD YYYY') >= current_date - interval '10 years'
```

-- 15. Find the top 10 actors who have appeared in the highest number of movies produced in India.
```
select*from netflix

select trim(unnest(string_to_array(casts, ',' ))) as Actor,
       count(*) as movies
	   from netflix
	   where type = 'Movie' and country like '%India%'
       group by 1
	   order by movies desc
       limit 10

```
	   

-- 16. Categorize the content based on the presence of the keywords 'kill' and 'violence' in the description field. 
     --Label content containing these keywords as 'Bad' and all other content as 'Good'. Count how many items fall into each category.
     
- using subquery
```

select category, 
       type, 
	   count(*)as total_content 
	   from (
                   SELECT *, ( case
                                    when description ilike '%kill%' or description ilike '%violence%' then 'bad'
                                    else 'good'
                                    end ) as category
                   from netflix
                 )

group by 1,2
order by total_content desc

```

- using cte
```
with cte as ( select count(*) as total_content,
                      ( case when description ilike '%kill%' or description ilike '%violence%' then 'Bad'
					   else 'Good'
					   end ) as category
					   from netflix
					   group by 2
            )
			select category, total_content 
			from cte
		    order by 2 desc
```

-- 17.  Total released content in the type for each country 
```

select count(*) AS total_content,
        trim(unnest(string_to_array(country,','))) as country ,
		type as category 
		from netflix
where country is not null and show_id is not null and type is not null
group by 2,3
order by 2 asc
```

-- 18. Most / total Release in each year and type.
```
select count(*) AS total_content,
       release_year as year ,
		type as category 
		from netflix
where country is not null and show_id is not null
group by 2,3
order by 2 desc
```
