# Analyzing South Seattle Violence via Open Data

I live in Columbia City in South Seattle and wanted to understand the recent perceived uptick in violence. Full analysis in [this blog post](http://lindsayrgwatt.com/blog/2017/07/analyzing-violence-in-south-seattle/).

This repo is set of data to help anyone get started if they want to repeat the analysis or try it for their neighborhood. Below I've listed some SQL I used to analyze data in Postgres with PostGIS; dumped output from it into [QGIS](http://www.qgis.org/en/site/) or this [Google Sheet](https://docs.google.com/spreadsheets/d/1cP0tpxhqyyH1mO9UbxK0e6EWRgPJzDf_wpNb-tfxzRg/edit?usp=sharing).

```
/* Import police data */
/* Source: https://data.seattle.gov/Public-Safety/Seattle-Police-Department-911-Incident-Response/3k2p-39jp/data */
CREATE TABLE police_calls
(
	event_clearance_description varchar,
	event_clearance_subgroup varchar,
	event_clearance_group varchar,
	event_clearance_data timestamp,
	hundred_block_location varchar,
	initial_type_description varchar,
	initial_type_subgroup varchar,
	initial_type_group varchar,
    latitude double precision,
    longitude double precision
);

/* Manually update headers; manually filter/reorder columns at data.seattle.gov */
COPY police_calls FROM '/Users/lindsayrgwatt/Downloads/Seattle_Police_Department_911_Incident_Response.csv' DELIMITER ',' CSV HEADER;

/* UPDATE police calls data to correct spatial reference projection */
/* Northern Washington spatial reference: https://epsg.io/32148 */
ALTER TABLE police_calls ADD COLUMN geom geometry(POINT,4326);
UPDATE police_calls SET geom = ST_SetSRID(ST_MakePoint(longitude,latitude),4326);
ALTER TABLE police_calls
 ALTER COLUMN geom TYPE geometry(Point,32148) 
  USING ST_Transform(geom,32148);

CREATE INDEX idx_police_geom ON police_calls USING GIST(geom);

/* Import neighborhod boundaries */
/* https://data.seattle.gov/dataset/Neighborhoods/2mbt-aqqx */

shp2pgsql -s SRID ~/Desktop/seattle_neighborhoods_32148.shp neighborhoods | psql -h localhost -d lindsayrgwatt -U lindsayrgwatt;

/* http://postgis.net/2013/08/30/tip_ST_Set_or_Transform/ */
/* Use QGIS to convert to 32148 and save as shapefile */
/* Set & convert to 32148 */
ALTER TABLE neighborhoods
 ALTER COLUMN geom TYPE geometry(MultiPolygon,32148) 
  USING ST_SetSRID(geom,32148);

/* Get counts of different offences */
SELECT
	event_clearance_description,
	COUNT (event_clearance_description),
	s_hood,
	l_hood
FROM
	police_calls
JOIN
	neighborhoods
ON ST_Contains(police_calls.geom, neighborhoods.geom)
WHERE
	event_clearance_description IN (
		'DRIVE BY SHOOTING (NO INJURIES)',
		'HOMICIDE',
		'PERSON WITH A GUN',
	)
GROUP BY
	event_clearance_description
ORDER BY
	event_clearance_description
;

/* Homicide and gun data */
SELECT
	police_calls.*,
	Extract(month from police_calls.event_clearance_date) AS month,
	Extract(year from police_calls.event_clearance_date) AS year,
	s_hood,
	l_hood
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'DRIVE BY SHOOTING (NO INJURIES)',
		'HOMICIDE',
		'PERSON WITH A GUN'
	)
ORDER BY
	police_calls.event_clearance_date DESC
;

/* Shots reported */
SELECT
	police_calls.*,
	Extract(month from police_calls.event_clearance_date) AS month,
	Extract(year from police_calls.event_clearance_date) AS year,
	s_hood,
	l_hood
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.initial_type_description LIKE '%SHOT%'
ORDER BY
	police_calls.event_clearance_date DESC
;

/* Gang-related */
SELECT
	police_calls.*,
	Extract(month from police_calls.event_clearance_date) AS month,
	Extract(year from police_calls.event_clearance_date) AS year,
	s_hood,
	l_hood
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'DISTURBANCE, GANG RELATED',
		'ASSAULTS, GANG RELATED',
		'ASSAULTS, GANG RELATED '
	)
;

/* Homicide and gun data in Columbia City */
SELECT
	police_calls.event_clearance_description,
	Extract(year from police_calls.event_clearance_date) AS year,
	Extract(month from police_calls.event_clearance_date) AS month,
	COUNT(police_calls.event_clearance_description)
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'DRIVE BY SHOOTING (NO INJURIES)'/*,
		'HOMICIDE',
		'PERSON WITH A GUN'*/
	) AND
	s_hood = 'Columbia City'
GROUP BY
	police_calls.event_clearance_description,
	month,
	year
ORDER BY
	police_calls.event_clearance_description,
	year DESC,
	month DESC
;

/* Shots reported in Columbia City */
SELECT
	Extract(year from police_calls.event_clearance_date) AS year,
	Extract(month from police_calls.event_clearance_date) AS month,
	COUNT(event_clearance_description)
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.initial_type_description LIKE '%SHOT%' AND
	s_hood = 'Columbia City'
GROUP BY
	month,
	year
ORDER BY
	year DESC,
	month DESC
;

/* Gang-related */
SELECT
	Extract(year from police_calls.event_clearance_date) AS year,
	Extract(month from police_calls.event_clearance_date) AS month,
	COUNT(event_clearance_description)
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'DISTURBANCE, GANG RELATED',
		'ASSAULTS, GANG RELATED',
		'ASSAULTS, GANG RELATED '
	) AND
	s_hood = 'Columbia City'
GROUP BY
	month,
	year
ORDER BY
	year DESC,
	month DESC
;

/* Assaults with a firearm */
SELECT
	Extract(year from police_calls.event_clearance_date) AS year,
	Extract(month from police_calls.event_clearance_date) AS month,
	COUNT(event_clearance_description)
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'ASSAULTS, FIREARM INVOLVED'
	) AND
	s_hood = 'Columbia City'
GROUP BY
	month,
	year
ORDER BY
	year DESC,
	month DESC
;

/* Gun calls */
SELECT
	police_calls.event_clearance_description,
	police_calls.event_clearance_subgroup
FROM
	police_calls
WHERE
	police_calls.event_clearance_subgroup = 'GUN CALLS'
GROUP BY
	police_calls.event_clearance_description,
	police_calls.event_clearance_subgroup
ORDER BY
	police_calls.event_clearance_description
;

/* Drive-by Shootings by Hour of Day in Columbia City */
SELECT
	Extract(year from police_calls.event_clearance_date) AS year,
	CASE Extract(hour from police_calls.event_clearance_date)
		WHEN 23 THEN 'NIGHT'
		WHEN 22 THEN 'NIGHT'
		WHEN 21 THEN 'DAY'
		WHEN 20 THEN 'DAY'
		WHEN 19 THEN 'DAY'
		WHEN 18 THEN 'DAY'
		WHEN 17 THEN 'DAY'
		WHEN 16 THEN 'DAY'
		WHEN 15 THEN 'DAY'
		WHEN 14 THEN 'DAY'
		WHEN 13 THEN 'DAY'
		WHEN 12 THEN 'DAY'
		WHEN 10 THEN 'DAY'
		WHEN 9 THEN 'DAY'
		WHEN 8 THEN 'DAY'
		WHEN 7 THEN 'DAY'
		WHEN 6 THEN 'DAY'
		WHEN 5 THEN 'NIGHT'
		WHEN 4 THEN 'NIGHT'
		WHEN 3 THEN 'NIGHT'
		WHEN 2 THEN 'NIGHT'
		WHEN 1 THEN 'NIGHT'
		WHEN 0 THEN 'NIGHT'
	END AS time_of_day,
	COUNT(police_calls.event_clearance_description)
FROM
	neighborhoods
JOIN
	police_calls
ON ST_Contains(neighborhoods.geom, police_calls.geom)
WHERE
	police_calls.event_clearance_description IN (
		'DRIVE BY SHOOTING (NO INJURIES)'
	) AND
	s_hood = 'Columbia City'
GROUP BY
	time_of_day,
	year
ORDER BY
	year DESC,
	time_of_day
;

```
