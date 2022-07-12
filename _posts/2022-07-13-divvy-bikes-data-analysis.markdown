---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_divvy
title: Data Wrangling (divvy bikes)

# post specific
# if not specified, .name will be used from _data/owner/[language].yml
author: Chris Arthur Rosaroso
# multiple category is not supported
category: projects
# multiple tag entries are possible
tags: [projects, documentation]
# thumbnail image for post
img: ":divvy.jpg"
# disable comments on this page
#comments_disable: true

# publish date
date: 2022-07-13 08:11:06 +0900

# seo
# if not specified, date will be used.
#meta_modify_date: 2022-02-10 08:11:06 +0900
# check the meta_common_description in _data/owner/[language].yml
#meta_description: ""

# optional
# if you enabled image_viewer_posts you don't need to enable this. This is only if image_viewer_posts = false
#image_viewer_on: true
# if you enabled image_lazy_loader_posts you don't need to enable this. This is only if image_lazy_loader_posts = false
#image_lazy_loader_on: true
# exclude from on site search
#on_site_search_exclude: true
# exclude from search engines
#search_engine_exclude: true
# to disable this page, simply set published: false or delete this file
#published: false
---

<!-- outline-start -->

<h1 align = "center">Data Wrangling</h1>

Data sources:
- [Divvy bikes](https://divvybikes.com), download the raw data-sets [here](https://divvy-tripdata.s3.amazonaws.com/index.html)
- [Chicago Data Portal](https://data.cityofchicago.org/), download the raw stations-table:
  - Download the updated version [here](https://data.cityofchicago.org/Transportation/Divvy-Bicycle-Stations/bbyy-e7gq).
  - Download the version I used [here](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_raw/Divvy_Bicycle_Stations.csv). June 28, 2022

> The Stations table continues to get updates. I noticed some changes since I downloaded the data on May 15, 2022 and redownload on June 28, 2022. When I check the site, there is an update on May 18, 2022 hence the change in data.

<sub>Tools used:</sub>
- *PostgreSQL*
- *RStudio*

<details><summary>Skills applied</summary>
<p>

- Git - version control
- Excel

  - Conditional Formatting
  - Data filter
  - VLOOKUP
  - Pivot table
  - IF/ IFNA
- SQL 

  - CREATE TABLE
  - CREATE DB
  - SELECT DISTINCT
  - UPDATE TABLE
  - HAVING
  - LIKE
  - WILDCARDS
  - ALTER TABLE/ ALTER COLUMN
  - IN
  - CASE
  - CAST
  - EXTRACT: YEAR, EPOCH
  - JOINS
  - UNION
  - SUB QUERY
  - ROUND (percentage)
- R

  -
  -
  -
  -
  -
- googling hard hehe
</p></details>

<h2 align = "center">Table of Content</h2>

I. [Setting up SQL Environment](#setting-up-sql-environment)

II. [Combining Data](#combining-data)

- [2013](#2013)
- [2014](#2014)
- [2015](#2015)
- [2016](#2016)
- [2017](#2017)
- [2018](#2018)
- [2019](#2019)
- [2020](#2020)
- [2021](#2021)

III. [Trips table](#trips-table)

- [First table: trips_p1](#first-table-trips_p1)
- [Second table: trips_p2](#second-table-trips_p2)
- [Combining tables: trips](#combine-table-trips)
- [NULL values](#null-values)

IV. [Stations table](#stations-table)

- [Missing stations](#missing-stations)
- [Cleaning](#cleaning)


<h2 align = "center">Setting up SQL Environment</h2>

The SQL database that I used in this analysis is [PostgreSQL](https://www.postgresql.org) and I used [pgAdmin 4](https://www.pgadmin.org/download/) as the database tool.

*PgAdmin 4* has a default schema named "Public", it can be seen under Servers > PostgreSQL 14 > Databases > postgres > Schemas > Public.

We need to create another schema named "bike_trips" to compile all our files under it, instead in "public" schema. Follow the steps:

1. *Right click* "Schemas".
2. *Click* "Create".
3. *Click* "Schema".
4. On **Name**, enter "bike_trips".
5. *Click* "Save".

Lastly, to open the **Query Tool** press *ALT + SHIFT + Q*, or click the Query tool icon on the upper-left corner of pgAdmin.

&nbsp;

<h2 align = "center">Combining data</h2>

The process of combining all the data into one table as a yearly data and using the **File Naming Convention (FNC)**.

Steps:

1. Download all the data for year 2013 to 2021 [here](https://divvy-tripdata.s3.amazonaws.com/index.html).
2. Download the stations data [here](https://data.cityofchicago.org/api/views/bbyy-e7gq/rows.csv?accessType=DOWNLOAD), this data is from [Chicago Data Portal](https://data.cityofchicago.org/). For this analysis, I will refer to the table as **Stations table**.
3. After extracting the zip files, compile separately all the yearly bike-trips data and the stations data included in that folder. Name the folder as the year it represents.
4. Start compiling the data.
5. For file naming convention, all combined trip-data must named by "year" followed by "-divvy-tripdata.csv"

*Using RStudio*
```r
# load the necessary library

install.library(tidyverse)
library(tidyverse)
```

&nbsp;

<h3 align = "center"><strong>2013</strong></h3>

Rename the trip data "Divvy-Trips_2013.csv" to "2013-divvy-tripdata.csv"

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2013 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2013 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2013-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER;
```

&nbsp;

<h3 align = "center"><strong>2014</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2014_tripdata
df_2014 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2014", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2014_tripdata
write.csv(df_2014,"D:/Github/large csv files/divvy-bikeshare/trips/2014-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2014-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2014 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2014 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2014-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2015</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2015_tripdata
df_2015 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2015", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2015_tripdata
write.csv(df_2015,"D:/Github/large csv files/divvy-bikeshare/trips/2015-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2015-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2015 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2015 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2015-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2016</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2016_tripdata
df_2016 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2016", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2016_tripdata
write.csv(df_2016,"D:/Github/large csv files/divvy-bikeshare/trips/2016-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2016-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2016 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
  ```

```sql
-- Import csv file
COPY bike_trips.trips_2016 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2016-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2017</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2017_tripdata
df_2017 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2017", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2017_tripdata
write.csv(df_2017,"D:/Github/large csv files/divvy-bikeshare/trips/2017-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2017-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2017 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2017 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2017-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2018</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2018_tripdata
df_2018 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2018", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2018_tripdata
write.csv(df_2018,"D:/Github/large csv files/divvy-bikeshare/trips/2018-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2018-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2018 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2018 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2018-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2019</strong></h3>

This data set has inconsistent column naming. By using [Notepad++](https://notepad-plus-plus.org/downloads/) to open large datasets, each quarterly files are opened and inspected. We can see that 2019_Q2 table has different column names than the other tables, which will affect the merging process of the csv files. 2019_Q2 column name was matched to the rest of the other quarterly tables.

<sub>*RStudio*</sub>

```r
# Merge 2019_tripdata
df_2019 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2019", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2019_tripdata
write.csv(df_2019,"D:/Github/large csv files/divvy-bikeshare/trips/2019-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2019-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2019 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2019 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2019-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2020</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2020_tripdata
df_2020 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2020", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2020_tripdata
write.csv(df_2020,"D:/Github/large csv files/divvy-bikeshare/trips/2020-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2020-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2020 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2020 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2020-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

&nbsp;

<h3 align = "center"><strong>2021</strong></h3>

<sub>*RStudio*</sub>

```r
# Merge 2021_tripdata
df_2021 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips/2021", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export the combined 2021_tripdata
write.csv(df_2021,"D:/Github/large csv files/divvy-bikeshare/trips/2021-divvy-tripdata.csv", row.names = FALSE)
```
> Follow the file naming guideline, file name should be "2021-divvy-tripdata.csv".

Import the csv file into the database.

<sub>*PostgreSQL*</sub>
```sql
-- Create table
CREATE TABLE bike_trips.trips_2021 (
  trip_id bigint, 
  start_time timestamp without time zone, 
  end_time timestamp without time zone, 
  bike_id int, 
  trip_duration int, 
  start_station_id int, 
  start_station_name varchar(50), 
  end_station_id int, 
  end_station_name varchar(50), 
  user_type text, 
  gender text, 
  birth_year int);
```

```sql
-- Import csv file
COPY bike_trips.trips_2021 (
  trip_id, 
  start_time, 
  end_time, 
  bike_id, 
  trip_duration, 
  start_station_id, 
  start_station_name, 
  end_station_id, 
  end_station_name, 
  user_type, 
  gender, 
  birth_year) 
FROM 'D:/Github/large csv files/divvy-bikeshare/trips/2021-divvy-tripdata.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

Upon checking the structure of the tables in each year, we can notice that 2013-2019 have same data structure, while 2020-2021 have a different type of structure.

Since the type of data from year 2020-2021 is different; where the station_ids are varchar instead of bigint and tables have an additional columns for stations' latitude and longitude, we will compile the two range of years separately.

Combine all the trips from year 2020-2021, and do the same for year 2020-2021.

> I named the year 2013-2019 table as trips_p1, and 2020-2021 table as trips_p2.

trips_p1
```sql
CREATE TABLE AS bike_trips.trips_p1
  SELECT * FROM bike_trips.trips_2013
  UNION ALL
  SELECT * FROM bike_trips.trips_2014
  UNION ALL
  SELECT * FROM bike_trips.trips_2015
  UNION ALL
  SELECT * FROM bike_trips.trips_2016
  UNION ALL
  SELECT * FROM bike_trips.trips_2017
  UNION ALL
  SELECT * FROM bike_trips.trips_2018
  UNION ALL
  SELECT * FROM bike_trips.trips_2019;
```

trips_p2
```sql
CREATE TABLE AS bike_trips.trips_p2
  SELECT * FROM bike_trips.trips_2020
  UNION ALL
  SELECT * FROM bike_trips.trips_2021;
```

&nbsp;

<h2 align = "center">Trips table</h2>

The process of cleaning station names and station IDs in trips table, by verifying and matching the data available in Google Maps and Stations table.

We need to import the **Stations** table to serve as a reference for our cleaning proces. But, before importing the file we need to make some changes:

Steps:
1. **Open Excel**: *Under Data* > *Get & Transform Data* > *Get Data* > *From File* > *From Text/CSV* > locate and import [Divvy_Bicycle_Stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_raw/Divvy_Bicycle_Stations.csv). Under *Data Type Detection* select *Do not detect data types* and click **Load**. Delete **Sheet1**.
2. **Remove row1**: Under *Table Design* > *Tools* > click *Convert to Range*, select **OK**. Then delete row1 which contains column#.
3. **Freeze the header row/ top row**: Under *View* > *Window* > *Freeze Panes* > *Freeze Top Row*.
4. **Delete duplicate stations**: Use conditional formatting on ColumnB (Station Name) to easily identify duplicate values. Under *Home* > *Styles* > *Conditional Formatting* > *Highlight Cell Rules* > *Duplicate Values*.
5. Sort sheet by **Station Name** and look for duplicate values. Delete the record which has larger station ID. Then proceed to the next process.
6. By using **Find and Replace**, remove all the parenthesis on **coordinates** column. Find "(" or ")" then leave the *Replace with* field as blank. Do it for both open and close parenthesis.
7. **Remove column**: **Docks in Service**
8. Change the values under **Status** column: Not in Service = No, In Service = Yes.
9. **Rename columns**: Station Name = **name**, Total Docks = **docks**, Status = **in_service**, and Location = **coordinate**.
12. Save as [Stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/Stations.csv).

*Import csv file*

```sql
-- Create table
CREATE TABLE bike_trips.stations (
  id bigint,
  name varchar,
  docks int,
  in_service text,
  latitude numeric,
  longitude numeric,
  coordinate point);
```

```sql
-- Import csv file
COPY bike_trips.stations (
  id,
  name,
  docks,
  in_service,
  latitude,
  longitude,
  coordinate)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/Stations.csv'
DELIMITER ',' CSV HEADER;
```

> P.S. You can open normally the csv file using Excel but it will mess-up the station IDs.

&nbsp;

<h3 align = "center"><strong>First table: trips_p1</strong></h3>

Run a query to find all the missing/ mismatch station names and IDs to the **Stations** table. This includes columns *start_station_name*, *end_station_name*, *start_station_id* and *end_station_id*.

```sql
-- stations in trips_p1 where not in Stations table, 2013-2019
-- check the data

SELECT t.id, t.name
FROM (
  SELECT DISTINCT start_station_id as id, start_station_name as name 
  FROM bike_trips.trips_p1
  UNION
  SELECT DISTINCT end_station_id as id, end_station_name as name
  FROM bike_trips.trips_p1) as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM bike_trips.stations as s
  WHERE s.id = t.id)
UNION
SELECT t.id, t.name
FROM (
  SELECT DISTINCT start_station_name as name, start_station_id as id
  FROM bike_trips.trips_p1
  UNION
  SELECT DISTINCT end_station_name as name, end_station_id as id
  FROM bike_trips.trips_p1) as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM bike_trips.stations as s
  WHERE s.name = t.name);

-- 163 records
```

Export the result as [trips_p1_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/trips_p1_stations.csv).

In order to analyze the table, the following is done.

<h4><strong>Process</strong>:</h4>

1. Open the csv file using MS Excel. Rename columns: **id = old_id** & **name = old_name**.
2. Create a new header for column C: **new_id**, column D: **new_name**, and column E: **changes**.
3. **Import the Stations table**: Under *Data* > *Get & Transform Data* > *Get Data* > *From File* > *From Text/CSV* > locate [Stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/Stations.csv). Under *Data Type Detection*, select *Do not detect data types* and click **Load**.
4. **Remove row1**: Under *Table Design* > *Tools* > click **Convert to Range**, select **OK**. Then delete row1 which contains column#.
5. Copy column **id** to the right of column **name**. *Right Click* column C and select Insert, a new empty column C should appear. Copy ColumnA to ColumnC.
6. **Freeze the header row/ top row**: Under *View* > *Window* > *Freeze Panes* > *Freeze Top Row*. Do the same for the other sheet.
7. In **trips_p1_stations** sheet:

   - **Cell C2**: enter the formula `=IFNA(VLOOKUP(TEXT(B2,0),Stations!$B:$D,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   - **Cell D2**: enter the formula `=IFNA(VLOOKUP(TEXT(A2,0),Stations!$A:$B,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   - **Cell E2**: enter the formula `=IF(AND(D2="same",C2="same"),"missing",IF(B2=D2,"same",IF(AND(A2<>C2,D2="same"),"id",IF(B2<>D2,"name"))))`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   
8. **Use conditinal formatting**: Under *Home* > *Styles* > *Conditional Formatting* > *Highlight Cell Rules* > ...

    - **Column C**: *Text that contains* > "*same*" with **Green Fill with Dark Green Text**.
    - **Column D**: *Text that contains* > "*same*" with **Green Fill with Dark Green Text**.
    - **Column E**: 
      - *Text that contains* > "*missing*" with **Light Red Fill with Dark Red Text**.
      - *Text that contains* > "*name*" with **Yellow Fill with Dark Yellow Text**.
      - *Text that contains* > "*id*" with **Green Fill with Dark Green Text**.
      
9. Validate the **new_name** column by searching each names in [Google Maps](https://www.google.com/maps) and locate nearby **divvy-stations**.

    - **Example**:

      <details><summary>Click Me!</summary>
      <p>

      ID = **17**, name = **Wood St & Division St**. Use [Google Maps](https://www.google.com/maps) to validate the station_name.

      - Search Chicago to focus the search in Chicago City.

      ![Chicago](https://snipboard.io/vYIsW9.jpg)

      - Enter the station_name, **Wood St & Division St**, and *press Enter*.
      - Click **Nearby** and search **divvy**, to search nearby divvy-bike stations.

      ![17](https://snipboard.io/n4qvdG.jpg)

      - Hover over to the nearest station to view the station name.

      ![17.2](https://snipboard.io/LHBqnm.jpg)

      The nearest station is **Honore St & Division St**.

      > Upon checking the **Stations** table on ID number 17, we can confirm that station_id 17 is Honore St & Division St.

      ![17 excel](https://snipboard.io/najrpI.jpg)

      > Thus, **Wood St & Division St** is a wrong station name and must be replaced with correct name **Honore St & Division St**. You can also notice under **changes** column, it says **name**, which means **name change**.

      ![sample excel](https://snipboard.io/ixN0TV.jpg)

      </p>
      </details>

&nbsp;

10. **Create another table**: 

    - For **missing stations**: (24 records)
      - Filter changes column with values **missing**. Copy columns **old_id** and **old_name** into a new blank workbook and save it as [missing_stations_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations_p1.csv).
    - For **id changes**: (2 records)
      - Filter changes column with values **id**. Copy columns **old_name** and **new_id** into a new blank workbook and save it as [id_changes_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/id_changes_p1.csv).
    - For **name changes**: (137 records)
      - Filter changes column with values **name**. Copy columns **old_name** and **new_name** into a new blank workbook and save it as [name_changes_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/name_changes_p1.csv).

> **Optional**: You can hide error values indicators. Under *File* > *Options* > *Formulas* > *Error Checking* > uncheck *Enable background error checking*.

> Upon checking, 2 station names from **trips_p1** table has already existing names in **Stations** table. Thus, duplicate station names with a different **station_id**.

&nbsp;  

Going back to PostgreSQL:

Stations names **Lakefront Trail & Bryn Mawr Ave** and **Michigan Ave & 71st St** already have existing IDs in **Stations** table.

Since the smaller station ID is not used in **Stations** table, I used it as a new station ID which are **459** and **651** for **Lakefront Trail & Bryn Mawr Ave** and **Michigan Ave & 71st St** respectively.

**SQL Queries:**

*Import csv file*

- id_changes_p1.csv
```sql
CREATE TABLE bike_trips.id_changes_p1 (
  old_name varchar, 
  new_id int);

-- Import
COPY bike_trips.id_changes_p1 (old_name, new_id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_p1.csv' 
DELIMITER ',' CSV HEADER;
```

- name_changes_p1.csv
```sql
-- Create table
CREATE TABLE bike_trips.name_changes_p1 (
  old_name varchar, 
  new_name varchar);

-- Import
COPY bike_trips.name_changes_p1 (old_name, new_name)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/name_changes_p1.csv' 
DELIMITER ',' CSV HEADER;
```

*ID change*

- start_station_id
```sql
UPDATE bike_trips.trips_p1 as s
SET start_station_id = c.new_id
FROM bike_trips.id_changes_p1 as c
WHERE s.start_station_name = c.old_name;
```

- end_station_id
```sql
UPDATE bike_trips.trips_p1 as s
SET end_station_id = c.new_id
FROM bike_trips.id_changes_p1 as c
WHERE s.end_station_name = c.old_name;
```

*Name change*
```sql
-- start_station_name
UPDATE bike_trips.trips_p1 as s
SET start_station_name = c.new_name
FROM bike_trips.name_changes_p1 as c
WHERE s.start_station_name = c.old_name;
```
```sql
-- end_station_name
UPDATE bike_trips.trips_p1 as s
SET end_station_name = c.new_name
FROM bike_trips.name_changes_p1 as c
WHERE s.end_station_name = c.old_name;
```

> P.S. After cleaning both tables, some data are added into **id_changes_p1.csv** & **name_changes_p1.csv**. Worry not since all the uploaded and linked files are updated.

&nbsp;

<h3 align = "center"><strong>Second table: trips_p2</strong></h3>

```sql
-- stations in trips_p2 where not in Stations table, 2020-2021
-- check the data
SELECT t.id, t.name
FROM (
  SELECT DISTINCT start_station_id as id, start_station_name as name 
  FROM bike_trips.trips_p2
  UNION
  SELECT DISTINCT end_station_id as id, end_station_name as name 
  FROM bike_trips.trips_p2) as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM bike_trips.stations as s
  WHERE CAST(s.id AS varchar) = t.id)
UNION
SELECT t.id, t.name
FROM (
  SELECT DISTINCT start_station_name as name, start_station_id as id 
  FROM bike_trips.trips_p2
  UNION
  SELECT DISTINCT end_station_name as name, end_station_id as id 
  FROM bike_trips.trips_p2)as t
WHERE NOT EXISTS (
  SELECT s.id, s.name
  FROM bike_trips.stations as s
  WHERE s.name = t.name);

-- 716 records
```
Export the result as [trips_p2_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/csv%20files/stations/trips_p2_stations.csv). 

**Process:**

1. Open the csv file using MS Excel. Rename columns: **id = old_id** & **name = old_name**.
2. Create a new header for column C: **new_id**, column D: **new_name** and column E: **changes**, and column F: **verified**.
3. **Import the Stations table**: Under *Data* > *Get & Transform Data* > *Get Data* > *From File* > *From Text/CSV* > locate [Stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/Stations.csv). Under *Data Type Detection* > select *Do not detect data types* and click **Load**.
4. **Remove row1**: Under *Table Design* > *Tools* > click **Convert to Range**, select **OK**. Then delete row1 which contains column#.
5. Copy column **id** to the right of column **name**. *Right Click* column C and select Insert, a new empty column C should appear. Copy ColumnA to ColumnC.
6. **Freeze the header row/ top row**: Under *View* > *Window* > *Freeze Panes* > *Freeze Top Row*. Do the same for the other sheet.
7. In **trips_p2_stations** sheet:

    - **Cell C2**: enter the formula `=IFNA(VLOOKUP(TEXT(B2,0),Stations!$B:$D,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the bottom-right corner of the cell to copy the formula in the entire row.
    - **Cell D2**: enter the formula `=IFNA(VLOOKUP(TEXT(A2,0),Stations!$A:$B,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the bottom-right corner of the cell to copy the formula in the entire row.
    - **Cell E2**: enter the formula `=IF(C2 = "", "missing", IF(AND(D2="same",C2="same"),"missing",IF(B2=D2,"same",IF(AND(A2<>C2,D2="same"),"id",IF(AND(B2<>D2,C2="same"),"name",IF(AND(A2<>C2,B2<>D2),"both"))))))`, then press Enter. Click cell E2 again and double-click the bottom-right corner of the cell to copy the formula in the entire row.

8. **Use conditinal formatting**: Under *Home* > *Styles* > *Conditional Formatting* > *Highlight Cell Rules* > ...

    - **Column A**: *Duplicate Values*.
    - **Column B**: *Duplicate Values*.
    - **Column C**: *Text that contains* > "*same*" with **Green Fill with Dark Green Text**.
    - **Column D**: *Text that contains* > "*same*" with **Green Fill with Dark Green Text**.
    - **Column E**:
      - *Text that contains* > "*missing*" with **Light Red Fill with Dark Red Text**.
      - *Text that contains* > "*name*" with **Yellow Fill with Dark Yellow Text**.
      - *Text that contains* > "*id*" with **Green Fill with Dark Green Text**.
      - *Text that contains* > "*both*" with **Custom Format**. I used **Light Blue fill with Dark Blue Text**.
    - **Column F**: *Text that contains* > "*y*" with **Green Fill with Dark Green Text**.
9. Validate the **new_name** column by searching each names in [Google Maps](https://www.google.com/maps) and locate nearby **divvy-stations**.

    <details><summary>Validating sample:</summary>
    <p>

    - station_id = 17
    - Either a missing station_id **17** or station_name **Wood St & Division St**.

    Using [Google Maps](https://www.google.com/maps) to validate the station_name.

    - Search **Chicago** to focus the search in Chicago City.

    ![Chicago](https://snipboard.io/vYIsW9.jpg)

    - Enter the station_name, **Wood St & Division St**, and *press Enter*.
    - Click **Nearby** and search **divvy**, to search nearby divvy-bike stations.

    ![17](https://snipboard.io/n4qvdG.jpg)

    - Hover over to the nearest station to view the station name.

    ![17.2](https://snipboard.io/LHBqnm.jpg)

    The nearest station is **Honore St & Division St**.

    > Upon checking the **Stations** table on ID number 17, we can confirm that station_id 17 is Honore St & Division St.

    ![17 excel](https://snipboard.io/najrpI.jpg)

    > Thus, **Wood St & Division St** is a wrong station name and must be replaced with correct name **Honore St & Division St**. You can also notice under **changes** column, it says **name**, which means **name change**.

    ![sample excel](https://snipboard.io/ixN0TV.jpg)

    </p></details>

&nbsp;  

10. **Cleaning process**: aside from `VLOOKUP`, we will use **Find**, **Google Maps**, and manual process to match the data.

    - **Example #1**: *Find*
      1. Filter changes column by **missing** and sort the sheet by **old_name** in ascending order.
      2. First row of **old_name** column is **Archer (Damen) Ave & 37th St** with ID = **18022**. Then, *Find* **Archer** in **Stations** sheet and click *Find all*.
      3. Among the results, we can find **Archer Damen Ave & 37th St** with ID = **645**.
      4. Go back to **trips_p2_stations** sheet. Replace the values in columns **new_id** and **new_name** with **645** and **Archer Damen Ave & 37th St** respectively.
      5. Notice than value in **changes** column changed from **missing** to **both**, which means that data needs to change its **name** and **id**.
      6. Put "**y**" in the **verified** column.

    - **Example #2**: *Google Maps*
      1. Filter changes column by **missing** and sort the sheet by **old_name** in ascending order.
      2. Second row of **old_name** column is **Base - 2132 W Hubbard Warehouse** with ID = **Hubbard Bike-checking (LBS-WH-TEST)**. The station_id is **varchar** data type, so we need to confirm the station name. If you remember, in **missing_stations_p1.csv** there is a missing station that contains **Hubbard**.
      3. Upon checking we can see ID = **671** and name = **HUBBARD ST BIKE CHECKING (LBS-WH-TEST)**. We should check if these two stations are the same by checking out its coordinate and locate it on **Google Maps**.
      4. **Run a query**:
      
          - Base - 2132 W Hubbard Warehouse
          ```sql
          SELECT start_lat, start_lng FROM bike_trips.trips_p2
          WHERE start_station_name = 'Base - 2132 W Hubbard Warehouse'

          /*
          start_lat = 41.8899545160741
          start_lng = -87.6806509494781
          coordinate = 41.8899545160741, -87.6806509494781
          */
          ```

          - HUBBARD ST BIKE CHECKING (LBS-WH-TEST)
          ```sql
          SELECT start_lat, start_lng FROM bike_trips.trips_p2
          WHERE start_station_name = 'HUBBARD ST BIKE CHECKING (LBS-WH-TEST)'

          /*
          start_lat = 41.89
          start_lng = -87.6807
          coordinate = 41.89, -87.6807
          */
          ```

      5. **Locate** both coordinates in Google Maps. 

      ![coordinate](https://snipboard.io/oHMIyk.jpg)

      6. Both coordinates point at **Divvy Service Warehouse**. Therefore, change the name of both stations to **Divvy Service Warehouse** and use ID = **671**.
      7. Put "**y**" in the **verified** column. 

    - **Example #3**: *Manual process*
      1. Sort sheet by column **id** in ascending order and look for duplicate values.
      2. On ID = **329**, we can see different records. One record has changes to **name** and the other has changes to **both**.

      ![329](https://snipboard.io/4ZGcKj.jpg)

      3. The station name **Central Park Ave & Douglas Blvd** is actually on the **Stations** table, its just the ID used in the **trips** table is different which creates a conflict with on station IDs. Therefore, we can change the value in **new_name** column by **same**.

      ![329 cleaned](https://snipboard.io/AqMiYN.jpg)

      4. Put "**y**" in the **verified** column.

     
11. **Create another table**: 

    - For **missing stations**:
      - Filter changes column with values **missing**. Copy columns **old_id**, **old_name**, and **new_id** into a new blank workbook. Remove row14 where **old_id** and **old_name** is equal to **NULL**. Remove all values in **old_id** column except if it has a "**same**" value on **new_id** column. Delete **new_id** column and save it as [missing_stations_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations_p2.csv).
    - For **id changes**:
      - Filter changes column with values **id** and **both**. Copy columns **old_name** and **new_id** into a new blank workbook. Remove duplicates on **old_name** column and remove rows where values on **old_name** column = **NULL**. Save it as [id_changes_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/id_changes_p2.csv).
    - For **name changes**:
      - Filter changes column with values **name** and **both**. Copy columns **old_name** and **new_name** into a new blank workbook. Remove duplicates on **old_name** column and remove rows where values on **old_name** column = **NULL**. Save it as [name_changes_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/name_changes_p2.csv).


> **Optional**: You can hide error values indicators. Under *File* > *Options* > *Formulas* > *Error Checking* > uncheck *Enable background error checking*.

> Upon checking we can identify which names are duplicated, used a wrong station_id and missing from the official **Stations** table. By sorting the column by **id_status**, we can see that all the station_id in column A has a mixed types of data, *varchar* and *bigint*.

> Some stations names have null values, data are filled after finding same station ID and vice-versa.

&nbsp;

**SQL Queries:**

*Import csv files*

- id_changes_p2.csv
```sql
-- Create table
CREATE TABLE bike_trips.id_changes_p2 (
  old_name varchar,
  new_id bigint);

-- Import
COPY bike_trips.id_changes_p2 (old_name, new_id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_p2.csv' 
DELIMITER ',' CSV HEADER NULL 'null';
```

- name_changes_p2.csv
```sql
-- Create table
CREATE TABLE bike_trips.name_changes_p2 (
  old_name varchar,
  new_name varchar);

-- Import
COPY bike_trips.name_changes_p2 (old_name, new_name)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/name_changes_p2.csv' 
DELIMITER ',' CSV HEADER NULL 'null';
```

*ID change*

- start_station_id
```sql
UPDATE bike_trips.trips_p2
SET start_station_id = CASE
  WHEN start_station_id = 'WL-008' THEN '57'
  WHEN start_station_id = '13221' THEN '61'
  WHEN start_station_id = '20215' THEN '732'
  END
WHERE start_station_id IN ('WL-008', '13221', '20215');
```
```sql
UPDATE bike_trips.trips_p2 as s
SET start_station_id = CAST(c.new_id AS varchar)
FROM bike_trips.id_changes_p2 as c
WHERE s.start_station_name = c.old_name;
```

- end_station_id
```sql
UPDATE bike_trips.trips_p2
SET end_station_id = CASE
  WHEN end_station_id = 'WL-008' THEN '57'
  WHEN end_station_id = '13221' THEN '61'
  WHEN end_station_id = '20215' THEN '732'
  END
WHERE end_station_id IN ('WL-008', '13221', '20215');
```
```sql
UPDATE bike_trips.trips_p2 as s
SET end_station_id = CAST(c.new_id AS varchar)
FROM bike_trips.id_changes_p2 as c
WHERE s.end_station_name = c.old_name;
```

*Name change*

- start_station
```sql
UPDATE bike_trips.trips_p2
SET start_station_name = CASE
  WHEN start_station_id = '57' THEN 'Clinton St & Roosevelt Rd'
  WHEN start_station_id = '61' THEN 'Wood St & Milwaukee Ave'
  WHEN start_station_id = '732' THEN 'Hegewisch Metra Station'
  END
WHERE start_station_id IN ('57', '61', '732');
```
```sql
UPDATE bike_trips.trips_p2 as s
SET start_station_name = c.new_name
FROM bike_trips.name_changes_p2 as c
WHERE s.start_station_name = c.old_name;
```

- end_station
```sql
UPDATE bike_trips.trips_p2
SET end_station_name = CASE
  WHEN end_station_id = '57' THEN 'Clinton St & Roosevelt Rd'
  WHEN end_station_id = '61' THEN 'Wood St & Milwaukee Ave'
  WHEN end_station_id = '732' THEN 'Hegewisch Metra Station'
  END
WHERE end_station_id IN ('57', '61', '732');
```
```sql
UPDATE bike_trips.trips_p2 as s
SET end_station_name = c.new_name
FROM bike_trips.name_changes_p2 as c
WHERE s.end_station_name = c.old_name;
```

> P.S. After cleaning both tables, some data are added into **id_changes_p1.csv** & **name_changes_p1.csv**. Worry not since all the uploaded and linked files are updated.

&nbsp;

<h3 align = "center"><strong>Combine table: trips</strong></h3>

Before combining both tables, some changes has to be made first. By checking the schema of the tables we can see the differences in their data. After modifying, merge both tables into one and save as **trips** table.

<h4><strong>Schema</strong></h4>

*Table Columns*

- trips_p1
```
| Field name         | Type                        |
| ------------------ | --------------------------- |
| trip_id            | bigint                      |
| start_time         | timestamp without time zone |
| end_time           | timestamp without time zone |
| bike_id            | integer                     |
| duration           | integer                     |
| start_station_id   | integer                     |
| start_station_name | character varying           |
| end_station_id     | integer                     |
| end_station_name   | character varying           |
| user_type          | text                        |
| gender             | text                        |
| birth_year         | integer                     |
```

- trips_p2
```
| Field name         | Type                        |
| ------------------ | --------------------------- |
| ride_id            | character varying           |
| rideable_type      | character varying           |
| start_time         | timestamp without time zone |
| end_time           | timestamp without time zone |
| start_station_id   | character varying           |
| start_station_name | character varying           |
| end_station_id     | character varying           |
| end_station_name   | character varying           |
| start_lat          | numeric                     |
| start_lng          | numeric                     |
| end_lat            | numeric                     |
| end_lng            | numeric                     |
| user_type          | text                        |
```

**Table changes**:
- *trips_p1*
  - change data type of column **trip_id**: *bigint* to *varchar*.
  - rename column: **trip_id** to **ride_id**
- **trips_p2**
  - remove columns: **start_lat**, **start_lng**, **end_lat** & **end_lng**.
  - fill missing data: **duration** by using start_time and end_time.
  - change data type of columns **start_station_id** & **end_station_id**: from *varchar* to *bigint*.

&nbsp;

**SQL Query:**

```sql
CREATE TABLE bike_trips.trips AS
SELECT 
  CAST(trip_id AS varchar) AS ride_id,
  CAST(null AS varchar) AS rideable_type,
  bike_id,
  start_time,
  end_time,
  trip_duration,
  CAST(start_station_id AS bigint),
  start_station_name,
  CAST(end_station_id AS bigint),
  end_station_name,
  user_type,
  gender,
  birth_year
FROM bike_trips.trips_p1
UNION ALL
SELECT
  ride_id,
  rideable_type,
  CAST(null AS int) AS bike_id,
  start_time,
  end_time,
  EXTRACT(EPOCH FROM(end_time - start_time))::int as trip_duration,
  CAST(start_station_id AS bigint),
  start_station_name,
  CAST(end_station_id AS bigint),
  end_station_name,
  user_type,
  CAST(null AS text) AS gender,
  CAST(null AS int) AS birth_year
FROM bike_trips.trips_p2;
```

<h3 align = "center"><strong>NULL values</strong></h3>

Some data are omitted in table **trips_p2** that has NULL **station_names** & **station_id**. Although these records doesn't contain any data about its *station_names* and *station_IDs*, these have a coordinates which then can be used later on to identify the nearby **DIVVY** station name. The data will be filled in the future after finishing my studies on Webscraping in Python.

*Number of records*
```
| Data     | No. of records |
| -------- | -------------- |
| trips_p1 |     24,426,783 |       
| trips_p2 |      9,136,746 |
| trips    |     33,563,529 |
| NULL     |      1,158,190 |
```

*Percent of NULL values*
```
| Table    | NULL values |
| -------- | ----------- |
| trips_p1 |         0 % |
| trips_p2 |     12.68 % |
| trips    |      3.45 % |
```

Divvy-bikeshare dataset contains 3.45% of NULL values. Since the percentage is small, we can then omit these records and proceed with the analysis.

&nbsp;    

**SQL Queries**:

*Number of records*
```sql
-- trips_p1: 24,426,783
SELECT COUNT(*) FROM bike_trips.trips_p1;

-- trips_p2: 9,136,746
SELECT COUNT(*) FROM bike_trips.trips_p2;

-- total: 33,563,529
SELECT COUNT(*) FROM bike_trips.trips;

-- total NULL records: 1,158,190
SELECT CAST(COUNT(*) AS numeric) 
  FROM bike_trips.trips_p2
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL);
```

*Percent of NULL values*

```sql
-- trips_p1
SELECT round(100 * 
 (SELECT CAST(COUNT(*) AS numeric) 
  FROM bike_trips.trips_p1
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM bike_trips.trips_p1)
			 , 2) AS NULL_values_percent

-- none
```

```sql
-- trips_p2
SELECT round(100 * 
 (SELECT CAST(COUNT(*) AS numeric) 
  FROM bike_trips.trips_p2
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM bike_trips.trips_p2)
			 , 2) AS NULL_values_percent

-- 12.68% of data is NULL
```

Since only table **trips_p2** has NULL values, we then use the table **trips_p2** as reference.
```sql
-- entire dataset
SELECT round(100 * 
 (SELECT CAST(COUNT(*) AS numeric) 
  FROM bike_trips.trips_p2
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM bike_trips.trips)
			 , 2) AS omitted_data_percent

-- 3.45% of data is NULL
```

&nbsp;

<h2 align = "center">Stations table</h2>

<h3><strong>Missing stations</strong></h3>

Now that we have the missing stations from the 2 tables, trips_p1 & trips_p2, combine them into one table and save as [missing_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations.csv).

*Import csv file*

- missing_stations.csv

Add the missing stations to Stations table
```sql
-- Import
COPY bike_trips.stations (
  id, 
  name, 
  docks,
  in_service, 
  latitude, 
  longitude, 
  coordinate)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/missing_stations.csv'
DELIMITER ',' CSV HEADER;
```


<h3><strong>Cleaning</strong></h3>

Some IDs need to change from the table.

*Import csv file*

- id_changes_stations.csv
```sql
-- Create table
CREATE TABLE bike_trips.id_changes_stations (
  name varchar,
  id bigint);
```
```sql
-- Import
COPY bike_trips.id_changes_stations (name, id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_stations.csv'
DELIMITER ',' CSV HEADER;
```

*ID change*

```sql
UPDATE bike_trips.stations as s
SET id = c.new_id
FROM bike_trips.id_changes_stations as c
WHERE s.name = c.old_name;
```

-- END

<!-- outline-end -->


