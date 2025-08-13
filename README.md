# Netflix Movies TV shows Data Analytics using SQL

![Netflix Logo](https://github.com/nandini-832/netflix_sql_project/blob/main/IMG_2187.PNG)

# Overview
This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL. The goal is to extract valuable insights and answer various business questions based on the dataset. The following README provides a detailed account of the project's objectives, business problems, solutions, findings, and conclusions.

# Objectives
Analyze the distribution of content types (movies vs TV shows).
Identify the most common ratings for movies and TV shows.
List and analyze content based on release years, countries, and durations.
Explore and categorize content based on specific criteria and keywords.
# Dataset
The data for this project is sourced from the Kaggle dataset: [
](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)

# Schema


```sql
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

-- 1. Identify the trend of content production — Count the number of titles released per year and visualize the growth or decline over time.

```


**`sql`**
```sql
SELECT 
    release_year,
    COUNT(*) AS total_titles
FROM netflix
GROUP BY release_year
ORDER BY release_year;
```

-- Objective: Analyze the yearly production trend by counting total titles released each year.


-- 2. Find the average number of actors per title — Measure whether movies or TV shows tend to have larger casts.

SELECT 
    type,
    ROUND(AVG(actor_count), 2) AS avg_actors_per_title
FROM (
    SELECT 
        type,
        title,
        CASE 
            WHEN COALESCE("casts", '') = '' THEN 0
            ELSE LENGTH("casts") - LENGTH(REPLACE("casts", ',', '')) + 1
        END AS actor_count
    FROM netflix
) AS sub
GROUP BY type;

-- Objective: Compare average cast size between movies and TV shows.


-- 3. Identify the top 5 fastest-growing genres in the last 5 years.

WITH recent_data AS (
    SELECT 
        release_year,
        TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre
    FROM netflix
    WHERE release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 5
),
year_range AS (
    SELECT 
        genre,
        MIN(release_year) AS first_year,
        MAX(release_year) AS last_year
    FROM recent_data
    GROUP BY genre
),
counts AS (
    SELECT 
        genre,
        release_year,
        COUNT(*) AS title_count
    FROM recent_data
    GROUP BY genre, release_year
)
SELECT 
    yr.genre,
    c1.title_count AS start_count,
    c2.title_count AS end_count,
    CASE 
        WHEN c1.title_count = 0 THEN NULL
        ELSE ROUND(((c2.title_count - c1.title_count) * 100.0 / c1.title_count), 2)
    END AS growth_percent
FROM year_range yr
JOIN counts c1 ON yr.genre = c1.genre AND yr.first_year = c1.release_year
JOIN counts c2 ON yr.genre = c2.genre AND yr.last_year = c2.release_year
ORDER BY growth_percent DESC NULLS LAST
LIMIT 5;

-- Objective: Identify genres with the highest percentage growth in titles over the last 5 years.


-- 4. Find the director who worked with the most unique actors and count them.

WITH director_list AS (
    SELECT 
        title,
        TRIM(UNNEST(STRING_TO_ARRAY("director", ','))) AS director_name
    FROM netflix
    WHERE "director" IS NOT NULL AND "director" <> ''
),
actor_list AS (
    SELECT 
        title,
        TRIM(UNNEST(STRING_TO_ARRAY("casts", ','))) AS actor_name
    FROM netflix
    WHERE "casts" IS NOT NULL AND "casts" <> ''
),
director_actor_pairs AS (
    SELECT DISTINCT
        d.director_name,
        a.actor_name
    FROM director_list d
    JOIN actor_list a ON d.title = a.title
)
SELECT 
    director_name,
    COUNT(DISTINCT actor_name) AS unique_actor_count
FROM director_actor_pairs
GROUP BY director_name
ORDER BY unique_actor_count DESC
LIMIT 1;

-- Objective: Determine the director who collaborated with the highest number of unique actors.


-- 5. Calculate the average IMDb-style rating proxy — use the rating field categories to estimate which content type generally has a more family-friendly audience.

SELECT 
    type,
    ROUND(AVG(rating_score), 2) AS avg_maturity_score
FROM (
    SELECT
        type,
        CASE
            WHEN rating = 'TV-Y' THEN 1
            WHEN rating = 'TV-Y7' THEN 2
            WHEN rating = 'TV-G' THEN 3
            WHEN rating = 'TV-PG' THEN 4
            WHEN rating = 'PG' THEN 5
            WHEN rating = 'PG-13' THEN 6
            WHEN rating = 'R' THEN 7
            WHEN rating = 'NC-17' THEN 8
            WHEN rating = 'TV-14' THEN 6
            WHEN rating = 'TV-MA' THEN 8
            ELSE NULL -- for NR or missing ratings
        END AS rating_score
    FROM netflix
    WHERE rating IS NOT NULL
) AS scored
WHERE rating_score IS NOT NULL
GROUP BY type
ORDER BY avg_maturity_score;

-- Objective: Estimate the average content maturity rating per content type to identify family-friendly content distribution.


-- 6. Find which month sees the most content added to Netflix (seasonal trend analysis).

SELECT 
    TO_CHAR(TO_DATE(date_added, 'Month DD, YYYY'), 'Month') AS month,
    COUNT(*) AS titles_added
FROM netflix
WHERE date_added IS NOT NULL AND date_added <> ''
GROUP BY month
ORDER BY titles_added DESC;

-- Objective: Analyze seasonal patterns by finding the month with the highest volume of new content additions.


-- 7. Identify the country with the highest ratio of TV shows to movies.

WITH country_content AS (
    SELECT
        TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS country_name,
        type
    FROM netflix
    WHERE country IS NOT NULL AND country <> ''
),
country_counts AS (
    SELECT
        country_name,
        COUNT(CASE WHEN type = 'TV Show' THEN 1 END) AS tv_show_count,
        COUNT(CASE WHEN type = 'Movie' THEN 1 END) AS movie_count
    FROM country_content
    GROUP BY country_name
    HAVING COUNT(CASE WHEN type = 'Movie' THEN 1 END) > 0 -- to avoid division by zero
)
SELECT
    country_name,
    tv_show_count,
    movie_count,
    ROUND(CAST(tv_show_count AS numeric) / movie_count, 2) AS tv_show_to_movie_ratio
FROM country_counts
ORDER BY tv_show_to_movie_ratio DESC
LIMIT 1;

-- Objective: Identify which country favors TV shows the most relative to movies based on content ratio.


-- 8. Find the actor who has worked across the highest number of different genres.

WITH actor_genres AS (
    SELECT
        TRIM(UNNEST(STRING_TO_ARRAY(casts, ','))) AS actor,
        TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre
    FROM netflix
    WHERE casts IS NOT NULL AND casts <> ''
      AND listed_in IS NOT NULL AND listed_in <> ''
),
actor_genre_count AS (
    SELECT
        actor,
        COUNT(DISTINCT genre) AS unique_genre_count
    FROM actor_genres
    GROUP BY actor
)
SELECT
    actor,
    unique_genre_count
FROM actor_genre_count
ORDER BY unique_genre_count DESC
LIMIT 1;

-- Objective: Discover the actor with the broadest range of genre experience.


-- 9. Determine which content has the shortest titles and which has the longest titles (text length analysis).
-- To find the shortest title(s):

SELECT title, LENGTH(title) AS title_length
FROM netflix
ORDER BY title_length ASC
LIMIT 5;

-- To find the longest title(s):

SELECT title, LENGTH(title) AS title_length
FROM netflix
ORDER BY title_length DESC
LIMIT 5;

-- Objective: Analyze title length extremes by finding the shortest and longest content titles.


-- 10. Find content that features both “love” and “war” in the description and analyze the year trend for such dual-themed shows/movies.

SELECT
    release_year,
    COUNT(*) AS count_dual_theme
FROM netflix
WHERE description ILIKE '%love%'
  AND description ILIKE '%war%'
GROUP BY release_year
ORDER BY release_year;

-- Objective: Track the yearly frequency of content featuring both “love” and “war” themes.


-- 11. Identify countries where more than 70% of produced content is documentaries.

WITH country_content AS (
    SELECT
        TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS country_name,
        CASE WHEN LOWER(listed_in) LIKE '%documentary%' THEN 1 ELSE 0 END AS is_documentary
    FROM netflix
    WHERE country IS NOT NULL AND country <> ''
),
country_stats AS (
    SELECT
        country_name,
        COUNT(*) AS total_count,
        SUM(is_documentary) AS documentary_count,
        SUM(is_documentary)::FLOAT / COUNT(*) AS documentary_ratio
    FROM country_content
    GROUP BY country_name
)
SELECT country_name, documentary_ratio
FROM country_stats
WHERE documentary_ratio > 0.7
ORDER BY documentary_ratio DESC;

-- Objective: Find countries dominated by documentary content (>70% share).


-- 12. Find the most common release year for movies with no listed director.

SELECT release_year, COUNT(*) AS count_movies
FROM netflix
WHERE type = 'Movie'
  AND (director IS NULL OR director = '')
GROUP BY release_year
ORDER BY count_movies DESC
LIMIT 1;

-- Objective: Identify the year with the highest number of director-less movie releases.


-- 13. Identify “one-hit wonder” directors — directors who have only one title on Netflix.

WITH director_list AS (
    SELECT 
        TRIM(UNNEST(STRING_TO_ARRAY(director, ','))) AS director_name
    FROM netflix
    WHERE director IS NOT NULL AND director <> ''
)
SELECT director_name, COUNT(*) AS title_count
FROM director_list
GROUP BY director_name
HAVING COUNT(*) = 1
ORDER BY director_name;

-- Objective: List directors with only one credited title (one-hit wonders).


-- 14. Determine the median number of seasons for TV shows by country.

WITH tv_shows AS (
    SELECT
        TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS country_name,
        CAST(SPLIT_PART(duration, ' ', 1) AS INTEGER) AS seasons
    FROM netflix
    WHERE type = 'TV Show'
      AND duration LIKE '%Season%'
      AND country IS NOT NULL AND country <> ''
)
SELECT
    country_name,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY seasons) AS median_seasons
FROM tv_shows
GROUP BY country_name
ORDER BY country_name;

-- Objective: Calculate median number of seasons for TV shows by country.


-- 15. Create a content maturity profile — categorize shows/movies as “Family”, “Teen”, or “Adult” based on the rating, then find the percentage share of each category per year.

WITH rating_category AS (
    SELECT
        release_year,
        CASE
            WHEN rating IN ('TV-Y', 'TV-Y7', 'TV-G', 'G', 'PG') THEN 'Family'
            WHEN rating IN ('PG-13', 'TV-PG', 'TV-14') THEN 'Teen'
            WHEN rating IN ('R', 'NC-17', 'TV-MA') THEN 'Adult'
            ELSE 'Unknown'
        END AS maturity_category
    FROM netflix
    WHERE release_year IS NOT NULL
),
yearly_counts AS (
    SELECT
        release_year,
        maturity_category,
        COUNT(*) AS count
    FROM rating_category
    GROUP BY release_year, maturity_category
),
yearly_totals AS (
    SELECT
        release_year,
        SUM(count) AS total_count
    FROM yearly_counts
    GROUP BY release_year
)
SELECT
    y.release_year,
    y.maturity_category,
    ROUND((y.count::DECIMAL / t.total_count) * 100, 2) AS percentage_share
FROM yearly_counts y
JOIN yearly_totals t ON y.release_year = t.release_year
WHERE y.maturity_category <> 'Unknown'
ORDER BY y.release_year, y.maturity_category;

-- Objective: Profile content maturity categories annually and determine their percentage share over time.
...

# Findings and Conclusions
Our analysis of Netflix’s content reveals several key insights. Content production trends fluctuate over time, with notable growth in certain genres in recent years. TV shows tend to have larger casts and vary in the number of seasons depending on the country.

Directors vary widely — some collaborate with many actors, while others have only one title on the platform. Content maturity ratings show Netflix balances family-friendly, teen, and adult programming, adjusting over time to audience preferences.

Seasonal spikes in new releases suggest strategic timing, and regional differences highlight that some countries focus more on TV shows or documentaries. Actors with diverse genre experience contribute to content variety, and title lengths vary significantly, reflecting different creative approaches.

Overall, these findings paint a picture of a dynamic, evolving content library shaped by audience demand, regional trends, and strategic decisions — offering valuable guidance for future content development and marketing strategies.

