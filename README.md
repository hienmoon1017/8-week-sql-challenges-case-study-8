# 8 Week SQL Challenges by Danny | Case Study #8 - Fresh Segments


### Data Exploration and Cleansing

```sql
UPDATE fresh_segments.interest_map
SET interest_summary = null
WHERE interest_summary = '';

-- update NULL values
UPDATE fresh_segments.interest_metrics
SET _month = case when _month = 'NULL' THEN null::integer
				 else _month::integer
				 end;

UPDATE fresh_segments.interest_metrics
SET _year = case when _year = 'NULL' then null ::integer
				else _year::integer
				end;

UPDATE fresh_segments.interest_metrics
SET month_year = null
WHERE month_year = 'NULL';

UPDATE fresh_segments.interest_metrics
SET interest_id = case when interest_id = 'NULL' then null::integer
						else interest_id::integer
						end;
ALTER TABLE fresh_segments.interest_metrics
ADD COLUMN id integer;

UPDATE fresh_segments.interest_metrics
SET id = interest_id::integer;
```
### 1. Update the fresh_segments.interest_metrics table by modifying the month_year column to be a date data type with the start of the month

```sql
UPDATE fresh_segments.interest_metrics
SET month_year = to_char(to_date(month_year,'YYYY-MM'),'MM-YYYY');
```
### 2. What is count of records in the fresh_segments.interest_metrics for each month_year value sorted in chronological order (earliest to latest) with the null values appearing first?

```sql
SELECT month_year
,count(*) as record_count
FROM fresh_segments.interest_metrics
GROUP BY 1
ORDER BY month_year is null DESC,
		 month_year;
```

_Result:_

![image](https://github.com/user-attachments/assets/7d9e1b48-e2ba-4798-a3e6-56c0996f042e)

### 3. What do you think we should do with these null values in the fresh_segments.interest_metrics

```sql
SELECT *
FROM fresh_segments.interest_metrics
WHERE month_year is null;

SELECT count(interest_id)
FROM fresh_segments.interest_metrics
WHERE month_year is null;
-- I decide to delele all rows having month_year is null
DELETE FROM fresh_segments.interest_metrics
WHERE month_year is null;

-- answer 3: should drop null values rows
```
### 4. How many interest_id values exist in the fresh_segments.interest_metrics table but not in the fresh_segments.interest_map table? What about the other way around?

```sql
SELECT count(distinct me.id) as count_id_not_in_map
FROM fresh_segments.interest_metrics me
LEFT JOIN fresh_segments.interest_map ma on ma.id = me.id
WHERE ma.id is null; -- check id not found in map

SELECT count(distinct ma.id) as count_id_not_in_metrics
FROM fresh_segments.interest_map ma
LEFT JOIN fresh_segments.interest_metrics me on ma.id = me.id
WHERE me.id is null; -- check id not found in metrics

SELECT ma.id
,ma.interest_name
,ma.interest_summary
FROM fresh_segments.interest_map ma
LEFT JOIN fresh_segments.interest_metrics me on ma.id = me.id
WHERE me.id is null; -- check id not found in metrics
```
### 5. Summarise the id values in the fresh_segments.interest_map by its total record count in this table

```sql
SELECT id
,count(id) as total_record
FROM fresh_segments.interest_map
GROUP BY 1
ORDER BY 2 DESC;
```
### 6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where interest_id = 21246 in your joined output and include all columns from fresh_segments.interest_metrics and all columns from fresh_segments.interest_map except from the id column.

```sql
-- should use interest_metrics LEFT JOIN interest_map to analyze data. Reason: zero id of interest_metrics not found in interest_map
SELECT me.id, me.month_year, me.composition, me.index_value, me.ranking, me.percentile_ranking
,ma.created_at, ma.last_modified, ma.interest_name, ma.interest_summary
FROM fresh_segments.interest_metrics me
LEFT JOIN fresh_segments.interest_map ma on ma.id = me.id
WHERE me.id = 21246;
```

_Result:_

![image](https://github.com/user-attachments/assets/4ee668e4-aa77-4897-a755-4d6e2d7f762d)

### 7. Are there any records in your joined table where the month_year value is before the created_at value from the fresh_segments.interest_map table? Do you think these values are valid and why?

```sql
SELECT count(distinct me.id) as records_before_created
FROM fresh_segments.interest_metrics me
LEFT JOIN fresh_segments.interest_map ma on ma.id = me.id
WHERE to_date(me.month_year,'MM-YYYY') < date(ma.created_at);
```

_Result:_

![image](https://github.com/user-attachments/assets/15ff1116-19d6-43d5-82ff-ed01ca869e8b)

### Interest Analysis

### 1. Which interests have been present in all month_year dates in our dataset?

```sql
SELECT ma.interest_name
,count(*) as counts
FROM fresh_segments.interest_metrics me
JOIN fresh_segments.interest_map ma on ma.id = me.id
GROUP BY 1
ORDER BY 2 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/ddebba05-8d0c-4382-82a8-ff3ccbe9ceb2)

### 2. Using this same total_months measure - calculate the cumulative percentage of all records starting at 14 months - which total_months value passes the 90% cumulative percentage value?

```sql
WITH cumulative_composition as
(
	SELECT month_year
	,sum(composition) over (order by month_year) as cumulative_compostion
	,row_number() over (order by month_year) as month_index
	FROM fresh_segments.interest_metrics	
)
,total_compositon as
(
	SELECT sum(composition) as composition_all_months
	FROM fresh_segments.interest_metrics
)
SELECT cc.month_year
,(cumulative_compostion / composition_all_months) * 100 as total_months
FROM cumulative_composition cc
CROSS JOIN total_compositon tc
WHERE cc.month_index >= 14
	and cumulative_compostion / composition_all_months * 100 >= 90
ORDER BY 2
LIMIT 1;
```

_Result:_


![image](https://github.com/user-attachments/assets/d4d5f8ad-8ca7-46af-8fe1-ba712b43fd72)

### 3.If we were to remove all interest_id values which are lower than the total_months value we found in the previous question - how many total data points would we be removing?

```sql
WITH cumulative_compostion as
(
	SELECT interest_id
	,sum(composition) over (order by month_year) as cumulative_composition
	,row_number() over (order by month_year) as month_index
	FROM fresh_segments.interest_metrics
)
,total_composition as
(
	SELECT sum(composition) as composition_all_months
	FROM fresh_segments.interest_metrics
)
SELECT count(distinct interest_id) as interest_id_count
FROM cumulative_compostion cc
CROSS JOIN total_composition tc
WHERE cc.month_index >= 14
	and (cumulative_composition / composition_all_months) *100 < 90;
```

_Result:_


![image](https://github.com/user-attachments/assets/ec47aea2-2c86-4212-887a-24efbfd0db07)

### 5. After removing these interests - how many unique interests are there for each month?

```sql
WITH interest_month as
(
	SELECT interest_id	
	,count(distinct month_year) as total_months_present
	FROM fresh_segments.interest_metrics
	GROUP BY 1
)
SELECT month_year
,count(distinct interest_id) as interest_id_count
FROM fresh_segments.interest_metrics
WHERE interest_id in (
						SELECT interest_id
						FROM interest_month
						WHERE total_months_present >= 14
					  )
GROUP BY 1
ORDER BY 1;
```

_Result:_


![image](https://github.com/user-attachments/assets/a950050c-e01d-4ddd-8f2d-745035adf378)

### Segment Analysis

### 1. Using our filtered dataset by removing the interests with less than 6 months worth of data, which are the top 10 and bottom 10 interests which have the largest composition values in any month_year? Only use the maximum composition value for each interest but you must keep the corresponding month_year

```sql
-- top 10 interest
WITH interest_month as
(
	SELECT interest_id
	,count(distinct month_year) as total_months_present
	FROM fresh_segments.interest_metrics
	GROUP BY 1
)
SELECT interest_id
,month_year
,composition
FROM fresh_segments.interest_metrics
WHERE interest_id in (
						SELECT interest_id
						FROM interest_month
						WHERE total_months_present >= 6
					  )
ORDER BY 3 DESC
LIMIT 10;
```

_Result:_


![image](https://github.com/user-attachments/assets/8406c6f2-ad46-426c-876e-8708f771b72a)

```sql
-- bottom 10 interest
WITH interest_month as
(
	SELECT interest_id
	,count(distinct month_year) as total_months_present
	FROM fresh_segments.interest_metrics
	GROUP BY 1
)
SELECT interest_id
,month_year
,composition
FROM fresh_segments.interest_metrics
WHERE interest_id in (
						SELECT interest_id
						FROM interest_month
						WHERE total_months_present >= 6
					  )
ORDER BY 3 ASC
LIMIT 10; 
```

_Result:_


![image](https://github.com/user-attachments/assets/fd69e667-3643-4e76-a722-af82a58eeb72)


### 2. Which 5 interests had the lowest average ranking value?

```sql
SELECT interest_id
,round(avg(ranking),2) as avg_ranking
FROM fresh_segments.interest_metrics
GROUP BY 1
ORDER BY 2 ASC
LIMIT 5;
```

_Result:_


![image](https://github.com/user-attachments/assets/b8108ce5-43c8-4d1d-9d8a-3c6db1821b7a)

### 3. Which 5 interests had the largest standard deviation in their percentile_ranking value?

```sql
SELECT interest_id
,round(stddev_pop(percentile_ranking)::numeric,2) as stddev_ranking --standard deviation
FROM fresh_segments.interest_metrics
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5;
```

_Result:_

![image](https://github.com/user-attachments/assets/33033dcb-370a-48d9-a24a-581ad1a14970)

### 4. For the 5 interests found in the previous question - what was minimum and maximum percentile_ranking values for each interest and its corresponding year_month value? Can you describe what is happening for these 5 interests?

```sql
SELECT interest_id
,month_year
,min(percentile_ranking) as min_percentile_ranking
,max(percentile_ranking) as max_percentile_ranking
FROM fresh_segments.interest_metrics
WHERE interest_id in ('6260', '23', '131', '150', '38992')
GROUP BY 1,2
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/260e9c61-1f90-46d7-91eb-8aaf2052465d)


### Index Analysis

### 1. What is the top 10 interests by the average composition for each month?

```sql
WITH ranking_interest as
(
	SELECT month_year
	,interest_id
	,row_number() over (partition by month_year order by composition / index_value DESC) as rank_avg_composition
FROM fresh_segments.interest_metrics
)
SELECT month_year
,interest_id
FROM ranking_interest
WHERE rank_avg_composition <= 10
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/ef441fa7-96a4-4bc5-b239-7fa3d4f6518b)

### 2. For all of these top 10 interests - which interest appears the most often?

```sql
WITH ranking_interest as
(
	SELECT month_year
	,interest_id
	,row_number() over (partition by month_year order by composition / index_value DESC) as rank_avg_composition
FROM fresh_segments.interest_metrics
)
SELECT interest_id
,count(*) as interest_count
FROM ranking_interest
WHERE rank_avg_composition <= 10
GROUP BY 1
ORDER BY 2 DESC
LIMIT 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/dfab9b16-0c8e-41fb-897b-6d3e3a3aa01b)



### 3. What is the average of the average composition for the top 10 interests for each month?

```sql
WITH ranking_interest as
(
	SELECT month_year
	,interest_id
	,composition / index_value as avg_composition
	,row_number() over (partition by month_year order by composition / index_value DESC) as rank_avg_composition
FROM fresh_segments.interest_metrics
)
SELECT month_year
,round(avg(avg_composition)::numeric,2) as avg_avg_composition
FROM ranking_interest
WHERE rank_avg_composition <= 10
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/e06a3b59-6a01-4109-b0fe-b9ee044c268d)


Thank you for stopping by, and I'm pleased to connect with you, my new friend!

Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
