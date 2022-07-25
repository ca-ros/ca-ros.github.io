<h1 align = "center">Data Wrangling</h1>

<!-- readme -->

<!-- outline-start -->

<h2 align = "center">Introduction</h2>

This dataset contains a list of biketrips in Chicago with trip duration greater than 60 seconds. It is provided by [Divvy bikes](https://divvybikes.com) according to the [Divvy Data License Agreement](https://ride.divvybikes.com/data-license-agreement).

<!-- outline-end -->

### 🚴‍♀️ The Data

Each trip is anonymized and includes:
- Trip start day and time
- Trip end day and time
- Trip start station
- Trip end station
- Rider type (Member, Single Ride, and Day Pass)

The data has been processed to remove trips that are taken by staff as they service and inspect the system; and any trips that were below 60 seconds in length (potentially false starts or users trying to re-dock a bike to ensure it was secure.)

Questions relating to trip data should be sent to <a href = "mailto:bike-data@lyft.com">bike-data@lyft.com</a>. Requests to use trademarks and trade names should be sent to <a href = "mailto:trademarks@lyft.com">trademarks@lyft.com</a>.

> Read [system-data](https://ride.divvybikes.com/system-data).

### 🔗 Data sources
- [Divvy bikes](https://divvybikes.com), download the raw data-sets [here](https://divvy-tripdata.s3.amazonaws.com/index.html)
- [Chicago Data Portal](https://data.cityofchicago.org/), download the raw stations-table:
  - Download the updated version [here](https://data.cityofchicago.org/Transportation/Divvy-Bicycle-Stations/bbyy-e7gq).
  - Download the version I used [here](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_raw/Divvy_Bicycle_Stations.csv). June 28, 2022

> The Stations table continues to get updates. I noticed some changes since I downloaded the data on May 15, 2022 and redownload on June 28, 2022. When I check the site, there is an update on May 18, 2022 hence the change in data.

&nbsp;

### Tools used
- [MS Excel](https://www.microsoft.com/en-ww/microsoft-365/excel)
- [Notepad++](https://notepad-plus-plus.org/downloads/)
- [PostgreSQL](https://www.postgresql.org/download/)
- [RStudio](https://www.rstudio.com/products/rstudio/download/)

<h2 align = "center">Table of Content</h2>

I. [Set up SQL Environment](#setting-up-sql-environment)

- [Create new schema](#create-new-schema)
- [Change the default schema](#change-the-default-schema)
- [Create a function](#create-a-function)

II. [Combining Data](#combining-data)

- [Data preparation](#data-preparation)
- [Load the libararies](#load-the-libraries)
- [Inspection](#inspection)
- [Merge files](#merge-files)
- [Import into the database](#import-into-the-database)
- [Data structure](#data-structure)

III. [Trips table](#trips-table)

- [Preparation](#preparation)
- [Import Stations table](#import-stations-table)
- [Enable Excel Macros](#enable-excel-macros)
- [First table: trips_p1](#first-table-trips_p1)
  - [Process](#process)
  - [Clean Macros](#clean-macros)
  - [SQL Query](#sql-query)
- [Second table: trips_p2](#second-table-trips_p2)
  - [Process](#process-1)
  - [SQL Query](#sql-query-1)
- [Combining tables: trips](#combine-table-trips)
  - [Schema](#schema)
  - [SQL Query](#sql-query-2)
- [NULL values](#null-values)
  - [SQL Query](#sql-query-3)

IV. [Stations table](#stations-table)

- [Missing stations](#missing-stations)
- [Cleaning](#cleaning)

V. [Cleaned Dataset](#cleaned-dataset)


<h2 align = "center" id = "setting-up-sql-environment">Set up SQL Environment</h2>

The SQL database that I used in this analysis is [PostgreSQL](https://www.postgresql.org) and I used [pgAdmin 4](https://www.pgadmin.org/download/) as the database tool.

*PgAdmin 4* has a default schema named "Public", it can be seen under Servers > PostgreSQL 14 > Databases > postgres > Schemas > Public.

<h3 id = "create-new-schema">Create new schema</h3>

We need another schema named "bike_trips" to compile all our files under it, instead in "public" schema. Follow the steps:

1. *Right click* "Schemas".
2. *Click* "Create".
3. *Click* "Schema".
4. On **Name**, enter "bike_trips".
5. *Click* "Save".

Lastly, to open the **Query Tool** press *ALT + SHIFT + Q*, or click the Query tool icon on the upper-left corner of pgAdmin.

<h3 id = "change-the-default-schema">Change the default schema</h3>

```sql
ALTER DATABASE postgres
SET search_path to bike_trips;

SET search_path to bike_trips;
```

<h3 id = "create-a-function">Create a function</h3>

We will use this everytime we create a table for yearly trips.

```sql
CREATE OR REPLACE FUNCTION trips_part(part varchar(30))
  RETURNS VOID
  LANGUAGE plpgsql AS
$func$
BEGIN
  EXECUTE format('
    CREATE TABLE IF NOT EXISTS %I (
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
      birth_year int
      )', 'trips_' || part);
END
$func$;
```

> To call a function, use `SELECT` e.g. `SELECT trips_part('p1')`.

&nbsp;

<h2 align = "center" id = "combining-data">Combining data</h2>

The process of aggregating all the data into one table. In preparation, data will be temporarily separated into two (2) parts due to differences in structure before merging into 1 table. Part 1 contains year 2013-2019 and part 2 contains year 2020-2021. 

<h3 id = "data-preparation">Data preparation</h3>

Steps:

1. Download all the data for year 2013 to 2021 [here](https://divvy-tripdata.s3.amazonaws.com/index.html).
2. Download the stations data [here](https://data.cityofchicago.org/api/views/bbyy-e7gq/rows.csv?accessType=DOWNLOAD), this data is from [Chicago Data Portal](https://data.cityofchicago.org/). For this analysis, I will refer to the table as **Stations table**.
3. After extracting the zip files, compile separately the bike-trips data and the stations data included in that folder. Name the folder as the part it represents *e.g. trips_p1*.
4. Start compiling the data, **trips_p1** folder must contain data from year *2013-2019* and **trips_p2** folder must contain data from year *2020-2021*.

<h3 id = "load-the-libraries">Load the libraries</h3>

*Using RStudio*

```r
install.library(tidyverse)
library(tidyverse)
```


<h3 id = "inspection">Inspection</h3>

Before merging the files, we should ensure the consistency of column names in each tables. To easily open large csv files we will be using [Notepad++](https://notepad-plus-plus.org/downloads/), each files are opened and inspected. Upon inspection, we saw that **Divvy_Trips_2019_Q2.csv** table has different column names than the rest, which will affect the merging process of the csv files. Thus, its column names was matched to the rest of the other tables.

<h3 id = "merge-files">Merge files</h3>

<sub>*RStudio*</sub>

```r
# Merge trips_p1
trips_p1 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips_p1", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Merge trips_p2
trips_p2 <- list.files(path="D:/Github/large csv files/divvy-bikeshare/trips_p2", full.names = TRUE) %>% 
  lapply(read_csv) %>% 
  bind_rows 
```

```r
# Export trips_p1
write.csv(trips_p1,"D:/Github/large csv files/divvy-bikeshare/trips_p1_raw.csv", row.names = FALSE)
```

```r
# Export trips_p2
write.csv(trips_p2,"D:/Github/large csv files/divvy-bikeshare/trips_p2_raw.csv", row.names = FALSE)
```

<h3 id = "import-into-the-database">Import into the database</h3>

We will use the SQL functions we create earlier.

<sub>*PostgreSQL*</sub>

```sql
-- Create table trips_p1
SELECT trips_part('p1');
```

```sql
-- Create table trips_p2
SELECT trips_part('p2');
```

```sql
-- Import p1
COPY trips_p1 (
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
FROM 'D:/Github/large csv files/divvy-bikeshare/trips_p1_raw.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

```sql
-- Import p2
COPY trips_p2 (
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
FROM 'D:/Github/large csv files/divvy-bikeshare/trips_p2_raw.csv' 
DELIMITER ',' CSV HEADER QUOTE '"' NULL 'NA';
```

<h3 id = "data-structure">Data structure</h3>

- trips_p1

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


- trips_p2

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



&nbsp;

<h2 align = "center" id = "trips-table">Trips table</h2>

The process of cleaning station names and station IDs in trips table, by verifying and matching the data available in Google Maps and Stations table. In this process we will be using Excel and macros to record and easily repeat the process.

We need to import the **Stations** table to serve as a reference for our cleaning proces. But, before importing the file we need to make some changes:

<h3 id = "preparation">Preparation</h3>

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

> P.S. You can open normally the csv file using Excel but it will mess-up the station IDs.

<h3 id = "import-stations-table">Import Stations table</h3>

```sql
-- Create table
CREATE TABLE stations (
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
COPY stations (
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

<h3 id = "enable-excel-macros">Enable Excel Macros</h3>

Excel macros is under Developer tab in Excel. The **Developer** tab isn't displayed by default, but you can add it to the ribbon.

Steps:
1. On the **File** tab, go to **Options** > **Customize Ribbon**.
2. Under **Customize the Ribbon** and under **Main tabs**, select the **Developer** check box.

After you show the tab, the Developer tab stays visible, unless you clear the check box or have to reinstall a Microsoft Office program.

The Developer tab is the place to go when you want to do or use the following:
- Write macros.
- Run macros that you previously recorded.
- Use XML commands.
- Use ActiveX controls.
- Create applications to use with Microsoft Office programs.
- Use form controls in Microsoft Excel.
- Work with the ShapeSheet in Microsoft Visio.
- Create new shapes and stencils in Microsoft Visio.

A macro is a series of commands that you can use to automate a repeated task, and can be run when you have to perform the task.

> Read [Show the Developer tab](https://support.microsoft.com/en-us/topic/show-the-developer-tab-e1192344-5e56-4d45-931b-e5fd9bea2d45) and [Macros in Office files](https://support.microsoft.com/en-us/office/macros-in-office-files-12b036fd-d140-4e74-b45e-16fed1a7e5c6).


&nbsp;

<h3 align = "center" id = "first-table-trips_p1"><strong>First table: trips_p1</strong></h3>

Run a query to find all the missing/ mismatch station names and IDs to the **Stations** table. This includes columns *start_station_name*, *end_station_name*, *start_station_id* and *end_station_id*.

```sql
-- stations in trips_p1 where not in Stations table, 2013-2019
-- check the data

WITH distinct_stations (id, name) AS
  (SELECT DISTINCT start_station_id as id, start_station_name as name 
  FROM trips_p1
  UNION
  SELECT DISTINCT end_station_id as id, end_station_name as name
  FROM trips_p1)
  
SELECT * FROM distinct_stations as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM stations as s
  WHERE s.id = t.id)
UNION
SELECT * FROM distinct_stations as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM stations as s
  WHERE s.name = t.name);

-- 163 records
```

Export the result as [trips_p1_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/trips_p1_stations.csv).

In order to analyze the table, the following is done.

<h4 id = "process">Process</h4>

1. Open the csv file using MS Excel, and save the workbook as "**trips_p1_stations.xlsm**".
2. **Start recording macro**: Under *Developer* > *Code* > *Record Macro*. Macro name: **divvy**, and click **OK**. 
3. Rename sheet to "**trips_stations**". Rename columns: **id = old_id** & **name = old_name**.
4. Create a new header for column C: **new_id**, column D: **new_name**, column E: **changes**, and column F: **verified**.
5. **Import the Stations table**: Under *Data* > *Get & Transform Data* > *Get Data* > *From File* > *From Text/CSV* > locate [Stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/Stations.csv). Then, under *Data Type Detection*, select *Do not detect data types* and click **Load**.
6. **Remove row1**: Under *Table Design* > *Tools* > click **Convert to Range**, select **OK**. Then delete row1 which contains column#.
7. Copy column **id** to the right of column **name**. *Right Click* column C and select Insert, a new empty column C should appear. Copy ColumnA to ColumnC.
8. **Freeze the header row/ top row**: Under *View* > *Window* > *Freeze Panes* > *Freeze Top Row*. Do the same for the other sheet.
9. In **trips_stations** sheet:

   - **Cell C2**: enter the formula `=IFNA(VLOOKUP(TEXT(B2,0),Stations!$B:$D,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   - **Cell D2**: enter the formula `=IFNA(VLOOKUP(TEXT(A2,0),Stations!$A:$B,2,FALSE),"same")`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   - **Cell E2**: enter the formula `=IF(C2 = "", "missing", IF(AND(D2="same",C2="same"),"missing",IF(B2=D2,"same",IF(AND(A2<>C2,D2="same"),"id",IF(AND(B2<>D2,C2="same"),"name",IF(AND(A2<>C2,B2<>D2),"both"))))))`, then press Enter. Click the cell again and double-click the *bottom-right* corner of the cell to copy the formula in the entire row.
   
10. **Use conditinal formatting**: Under *Home* > *Styles* > *Conditional Formatting* > *Highlight Cell Rules* > ...

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

11. **End macro recording**: Under *Developer* > *Code* > *Stop Recording*.
12. Validate the **new_name** column by searching each names in [Google Maps](https://www.google.com/maps) and locate nearby **divvy-stations**.

    <div id = "validating-sample">Validating sample:</div>

    - station_id = **17**, station_name = Wood St & Division St

    > Either a missing station_id or station_name. 
      
    Using [Google Maps](https://www.google.com/maps) to validate the station_name.

      - Search Chicago to focus the search in Chicago City.

      ![Chicago](https://snipboard.io/vYIsW9.jpg)

      - Enter the station_name, **Wood St & Division St**, and *press Enter*.
      - Click **Nearby** and search **divvy**, to search nearby divvy-bike stations.

      ![17](https://snipboard.io/n4qvdG.jpg)

      - Hover over to the nearest station to view the station name.

      ![17.2](https://snipboard.io/LHBqnm.jpg)

      The nearest station is **Honore St & Division St**.

      > Upon checking the **Stations** table on ID number 17, we can confirm that station_id 17 is Honore St & Division St.

      ![17 excel](https://snipboard.io/9MqQhg.jpg)

      > Thus, **Wood St & Division St** is a wrong station name and must be replaced with correct name **Honore St & Division St**. You can also notice under **changes** column, it says **name**, which means **name change**.

      ![sample excel](https://snipboard.io/tHhYAR.jpg)

13. **Create new tables**

    - **Recording Macro**

      1. **Start macro recording**: Macro name: **newtables**, and click **OK**.
      2. Filter **trips_stations** sheet by **changes** column with values:

          - "**missing**". Copy columns **old_id** and **old_name** into a new sheet and rename it to **missing_stations**.
          - "**id**". Copy columns **old_name** and **new_id** into a new sheet and rename it to **id_changes**.
          - "**name**". Copy columns **old_name** and **new_name** into a new sheet and rename it to **name_changes**.
      3. **End macro recording**.
  
    - For **missing stations**: (24 records)

      1. **Create new table**: Right click sheet **missing_stations** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [missing_stations_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations_p1.csv).

    - For **id changes**: (2 records)

      1. **Create new table**: Right click sheet **id_changes** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [id_changes_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/id_changes_p1.csv).

    - For **name changes**: (137 records)

      1. **Create new table**: Right click sheet **name_changes** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [name_changes_p1.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/name_changes_p1.csv).

14. **Save** the workbook, [trips_p1_stations.xlsm](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/trips_p1_stations.xlsm).

> **Optional**: You can hide error values indicators. Under *File* > *Options* > *Formulas* > *Error Checking* > uncheck *Enable background error checking*.

> Upon checking, 2 station names from **trips_p1** table has already existing names in **Stations** table. Thus, duplicate station names with a different **station_id**.

<h4 id = "clean-macros">Clean Macros</h4>
#### Clean Macros

We need to debug and clean the macros to remove any possible errors. Not only this will remove errors, the code will also be easy to read. Open the Visual Basic: Under *Developer* > *Code* > *Visual Basic*. Under the (*Workbook name*) > *Modules* > *Module1*. The macros recorded must be the same as the codes below. 

You can also just import the modules. On Microsoft VBA, right-click *Modules* folder > *Import file* > locate the modules > click *Open*. <br>

Download the modules here:
- [divvy.bas](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/vba/divvy.bas)
- [newtables.bas](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/vba/newtables.bas)

**VBScripts**:

- divvy.bas

```vb
Sub divvy()
'
' divvy Macro
'
' Keyboard Shortcut: Ctrl+Shift+Y
'
    ActiveSheet.Name = "trips_stations"
    Range("A1").Value = "old_id"
    Range("B1").Value = "old_name"
    Range("C1").Value = "new_id"
    Range("D1").Value = "new_name"
    Range("E1").Value = "changes"
    Range("F1").Value = "verified"
    ActiveWorkbook.Queries.Add Name:="Stations", Formula:= _
        "let" & Chr(13) & "" & Chr(10) & "    Source = Csv.Document(File.Contents(""C:\Users\Chris\Documents\GitHub\divvy-bikeshare\data wrangling\csv files\stations_cleaning\Stations.csv""),[Delimiter="","", Columns=7, Encoding=1252, QuoteStyle=QuoteStyle.None])," & Chr(13) & "" & Chr(10) & "    #""Change Type"" = Table.TransformColumnTypes(Source,{{""Column1"", type text}, {""Column2"", type text}, {""Column3"", type text}, {""" & _
        "Column4"", type text}, {""Column5"", type text}, {""Column6"", type text}, {""Column7"", type text}})" & Chr(13) & "" & Chr(10) & "in" & Chr(13) & "" & Chr(10) & "    #""Change Type"""
    ActiveWorkbook.Worksheets.Add
    With ActiveSheet.ListObjects.Add(SourceType:=0, Source:= _
        "OLEDB;Provider=Microsoft.Mashup.OleDb.1;Data Source=$Workbook$;Location=Stations;Extended Properties=""""" _
        , Destination:=Range("$A$1")).QueryTable
        .CommandType = xlCmdSql
        .CommandText = Array("SELECT * FROM [Stations]")
        .RowNumbers = False
        .FillAdjacentFormulas = False
        .PreserveFormatting = True
        .RefreshOnFileOpen = False
        .BackgroundQuery = True
        .RefreshStyle = xlInsertDeleteCells
        .SavePassword = False
        .SaveData = True
        .AdjustColumnWidth = True
        .RefreshPeriod = 0
        .PreserveColumnInfo = True
        .ListObject.DisplayName = "Stations"
        .Refresh BackgroundQuery:=False
    End With
    ActiveSheet.Name = "Stations"
    ActiveSheet.ListObjects("Stations").Unlist
    Application.CommandBars("Queries and Connections").Visible = False
    Rows("1:1").Select
    Selection.Delete Shift:=xlUp
    Cells.Style = "Normal"
    Rows("1:1").Font.Bold = True
    Columns("C:C").Select
    Selection.Insert Shift:=xlToRight, CopyOrigin:=xlFormatFromLeftOrAbove
    Columns("A:A").Copy
    Range("C1").Select
    ActiveSheet.Paste
    Application.CutCopyMode = False
    Range("A1").Select
    Columns("A:A").ColumnWidth = 13.43
    Columns("B:B").ColumnWidth = 34
    Columns("C:C").ColumnWidth = 8.57
    Rows("1:1").Select
    Selection.AutoFilter
    Sheets("trips_stations").Select
    Rows("1:1").Select
    Selection.Font.Bold = True
    With ActiveWindow
        .SplitColumn = 0
        .SplitRow = 1
    End With
    ActiveWindow.FreezePanes = True
    Selection.AutoFilter
    Columns("B:B").ColumnWidth = 36.14
    Columns("C:C").EntireColumn.AutoFit
    Columns("D:D").ColumnWidth = 34.29
    Columns("E:E").EntireColumn.AutoFit
    Columns("F:F").EntireColumn.AutoFit
    ActiveCell.FormulaR1C1 = "old_id"
    Range("C2").Select
    With Selection
        .Formula = "=IFNA(VLOOKUP(TEXT(B2,0),Stations!$B:$D,2,FALSE),""same"")"
        .AutoFill Destination:=Range("C2:C" & Range("A" & Rows.Count).End(xlUp).Row)
    End With
    Range(Selection, Selection.End(xlDown)).Select
    Range("D2").Select
    With Selection
        .Formula = "=IFNA(VLOOKUP(TEXT(A2,0),Stations!$A:$B,2,FALSE),""same"")"
        .AutoFill Destination:=Range("D2:D" & Range("A" & Rows.Count).End(xlUp).Row)
    End With
    Range(Selection, Selection.End(xlDown)).Select
    Range("E2").Select
    With Selection
        .Formula = "=IF(C2 = """", ""missing"", IF(AND(D2=""same"",C2=""same""),""missing"",IF(B2=D2,""same"",IF(AND(A2<>C2,D2=""same""),""id"",IF(AND(B2<>D2,C2=""same""),""name"",IF(AND(A2<>C2,B2<>D2),""both""))))))"
        .AutoFill Destination:=Range("E2:E" & Range("A" & Rows.Count).End(xlUp).Row)
    End With
    Range(Selection, Selection.End(xlDown)).Select
    Columns("A:A").Select
    Selection.FormatConditions.AddUniqueValues
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    Selection.FormatConditions(1).DupeUnique = xlDuplicate
    With Selection.FormatConditions(1).Font
        .Color = -16383844
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13551615
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("B:B").Select
    Selection.FormatConditions.AddUniqueValues
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    Selection.FormatConditions(1).DupeUnique = xlDuplicate
    With Selection.FormatConditions(1).Font
        .Color = -16383844
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13551615
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("F:F").Select
    Selection.FormatConditions.Add Type:=xlTextString, String:="y", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16752384
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13561798
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("C:C").Select
    Selection.FormatConditions.Add Type:=xlTextString, String:="same", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16752384
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13561798
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("D:D").Select
    Selection.FormatConditions.Add Type:=xlTextString, String:="same", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16752384
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13561798
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("E:E").Select
    Selection.FormatConditions.Add Type:=xlTextString, String:="missing", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16383844
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13551615
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Selection.FormatConditions.Add Type:=xlTextString, String:="name", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16754788
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 10284031
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Selection.FormatConditions.Add Type:=xlTextString, String:="id", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .Color = -16752384
        .TintAndShade = 0
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .Color = 13561798
        .TintAndShade = 0
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Columns("E:E").Select
    Selection.FormatConditions.Add Type:=xlTextString, String:="both", _
        TextOperator:=xlContains
    Selection.FormatConditions(Selection.FormatConditions.Count).SetFirstPriority
    With Selection.FormatConditions(1).Font
        .ThemeColor = xlThemeColorAccent1
        .TintAndShade = -0.499984740745262
    End With
    With Selection.FormatConditions(1).Interior
        .PatternColorIndex = xlAutomatic
        .ThemeColor = xlThemeColorAccent1
        .TintAndShade = 0.399945066682943
    End With
    Selection.FormatConditions(1).StopIfTrue = False
    Sheets("trips_stations").Move Before:=Sheets(1)
    Range("A1").Select
End Sub
```

- newtables

```vb
Sub newtables()
'
' newtables Macro
'

'
    ActiveSheet.Range("$A$1:$F" & Range("A" & Rows.Count).End(xlUp).Row).AutoFilter Field:=5, Criteria1:="missing"
    Range("A:A,B:B").Select
    Range("B1").Activate
    Selection.Copy
    Sheets.Add After:=ActiveSheet
    ActiveSheet.Paste
    Application.CutCopyMode = False
    Columns("A:A").ColumnWidth = 36.43
    Columns("B:B").ColumnWidth = 33.14
    ActiveSheet.Name = "missing_stations"
    Range("A1").Select
    Sheets("trips_stations").Select
    ActiveSheet.Range("$A$1:$F" & Range("A" & Rows.Count).End(xlUp).Row).AutoFilter Field:=5, Criteria1:="=both", _
        Operator:=xlOr, Criteria2:="=id"
    Range("B:B,C:C").Select
    Range("C1").Activate
    Selection.Copy
    Sheets.Add After:=ActiveSheet
    ActiveSheet.Paste
    Application.CutCopyMode = False
    Columns("A:A").ColumnWidth = 36.43
    Columns("B:B").ColumnWidth = 33.14
    ActiveSheet.Name = "id_changes"
    Range("A1").Select
    Sheets("trips_stations").Select
    ActiveSheet.Range("$A$1:$F" & Range("A" & Rows.Count).End(xlUp).Row).AutoFilter Field:=5, Criteria1:="=both", _
        Operator:=xlOr, Criteria2:="=name"
    Range("B:B,D:D").Select
    Range("D1").Activate
    Selection.Copy
    Sheets.Add After:=ActiveSheet
    ActiveSheet.Paste
    Application.CutCopyMode = False
    Columns("A:A").ColumnWidth = 36.43
    Columns("B:B").ColumnWidth = 33.14
    ActiveSheet.Name = "name_changes"
    Range("A1").Select
    Sheets("trips_stations").Select
    Range("A1").Select
    ActiveSheet.Range("$A$1:$F$164").AutoFilter Field:=5
End Sub
```

<h4 id = "sql-query">SQL Query</h4>

Stations names **Lakefront Trail & Bryn Mawr Ave** and **Michigan Ave & 71st St** already have existing IDs in **Stations** table.

Since the smaller station ID is not used in **Stations** table, I used it as a new station ID which are **459** and **651** for **Lakefront Trail & Bryn Mawr Ave** and **Michigan Ave & 71st St** respectively.

*Import csv file*

- id_changes_p1.csv

```sql
CREATE TABLE id_changes_p1 (
  old_name varchar, 
  new_id int);

-- Import
COPY id_changes_p1 (old_name, new_id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_p1.csv' 
DELIMITER ',' CSV HEADER;
```

- name_changes_p1.csv

```sql
-- Create table
CREATE TABLE name_changes_p1 (
  old_name varchar, 
  new_name varchar);

-- Import
COPY name_changes_p1 (old_name, new_name)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/name_changes_p1.csv' 
DELIMITER ',' CSV HEADER;
```

*ID change*

- start_station_id

```sql
UPDATE trips_p1 as s
SET start_station_id = c.new_id
FROM id_changes_p1 as c
WHERE s.start_station_name = c.old_name;
```

- end_station_id

```sql
UPDATE trips_p1 as s
SET end_station_id = c.new_id
FROM id_changes_p1 as c
WHERE s.end_station_name = c.old_name;
```

*Name change*

```sql
-- start_station_name
UPDATE trips_p1 as s
SET start_station_name = c.new_name
FROM name_changes_p1 as c
WHERE s.start_station_name = c.old_name;
```

```sql
-- end_station_name
UPDATE trips_p1 as s
SET end_station_name = c.new_name
FROM name_changes_p1 as c
WHERE s.end_station_name = c.old_name;
```

> P.S. After cleaning both tables, some data are added into **id_changes_p1.csv** & **name_changes_p1.csv**. Worry not since all the uploaded and linked files are updated.

&nbsp;

<h3 align = "center" id = "second-table-trips_p2"><strong>Second table: trips_p2</strong></h3>

```sql
-- stations in trips_p2 where not in Stations table, 2020-2021
-- check the data

WITH distinct_stations (id, name) AS
  (SELECT DISTINCT start_station_id as id, start_station_name as name 
  FROM trips_p2
  UNION
  SELECT DISTINCT end_station_id as id, end_station_name as name
  FROM trips_p2)
  
SELECT * FROM distinct_stations as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM stations as s
  WHERE CAST(s.id AS varchar) = t.id)
UNION
SELECT * FROM distinct_stations as t
WHERE NOT EXISTS (
  SELECT id, name
  FROM stations as s
  WHERE s.name = t.name);

-- 716 records
```

Export the result as [trips_p2_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/csv%20files/stations/trips_p2_stations.csv). 

<h4 id = "process-1">Process</h4>

1. Open the csv file using MS Excel, amd save the workbook as "trips_p2_stations.xlsm". 
2. **Import macros**: Under *Developer* > *Code* > *Visual Basic*. Right-click *VBAProject* > *Import File* > upload files **divvy.bas** and **newtables.bas**.
3. **Run divvy module**: Under *Modules* folder > open *divvy* > and click **Run** or **press F5**.
4. Validate the **new_name** column by searching each names in [Google Maps](https://www.google.com/maps) and locate nearby **divvy-stations**.

    - Refer to earlier [sample](#validating-sample)

5. **Cleaning process**: aside from `VLOOKUP`, we will use **Find**, **Google Maps**, and manual process to match the data.

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
          SELECT start_lat, start_lng FROM trips_p2
          WHERE start_station_name = 'Base - 2132 W Hubbard Warehouse'

          /*
          start_lat = 41.8899545160741
          start_lng = -87.6806509494781
          coordinate = 41.8899545160741, -87.6806509494781
          */
          ```

          - HUBBARD ST BIKE CHECKING (LBS-WH-TEST)

          ```sql
          SELECT start_lat, start_lng FROM trips_p2
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

     
6. **Create another table**: 

    - **Run Macro**: Open *Visual Basic*. Under Modules folder > open **newtables** and press **F5**.

    - For **missing stations**:

      1. **Create new table**: Right click sheet **missing_stations** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [missing_stations_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations_p2.csv).

    - For **id changes**: (2 records)

      1. **Create new table**: Right click sheet **id_changes** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [id_changes_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/id_changes_p2.csv).

    - For **name changes**: (137 records)

      1. **Create new table**: Right click sheet **name_changes** > *Move or Copy* > *To book* > *new book*.
      2. Save it as [name_changes_p2.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/name_changes_p2.csv).

7. **Save** the workbook, [trips_p2_stations.xlsm](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/trips_p2_stations.xlsm).

> **Optional**: You can hide error values indicators. Under *File* > *Options* > *Formulas* > *Error Checking* > uncheck *Enable background error checking*.

> Upon checking we can identify which names are duplicated, used a wrong station_id and missing from the official **Stations** table. By sorting the column by **id_status**, we can see that all the station_id in column A has a mixed types of data, *varchar* and *bigint*.

> Some stations names have null values, data are filled after finding same station ID and vice-versa.

&nbsp;

<h4 id = "sql-query-1">SQL Query</h4>

*Import csv files*

- id_changes_p2.csv

```sql
-- Create table
CREATE TABLE id_changes_p2 (
  old_name varchar,
  new_id bigint);
```

```sql
-- Import
COPY id_changes_p2 (old_name, new_id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_p2.csv' 
DELIMITER ',' CSV HEADER NULL 'null';
```

- name_changes_p2.csv

```sql
-- Create table
CREATE TABLE name_changes_p2 (
  old_name varchar,
  new_name varchar);
```

```sql
-- Import
COPY name_changes_p2 (old_name, new_name)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/name_changes_p2.csv' 
DELIMITER ',' CSV HEADER NULL 'null';
```

*ID change*

- start_station_id

```sql
UPDATE trips_p2
SET start_station_id = CASE
  WHEN start_station_id = 'WL-008' THEN '57'
  WHEN start_station_id = '13221' THEN '61'
  WHEN start_station_id = '20215' THEN '732'
  END
WHERE start_station_id IN ('WL-008', '13221', '20215');
```

```sql
UPDATE trips_p2 as s
SET start_station_id = CAST(c.new_id AS varchar)
FROM id_changes_p2 as c
WHERE s.start_station_name = c.old_name;
```

- end_station_id

```sql
UPDATE trips_p2
SET end_station_id = CASE
  WHEN end_station_id = 'WL-008' THEN '57'
  WHEN end_station_id = '13221' THEN '61'
  WHEN end_station_id = '20215' THEN '732'
  END
WHERE end_station_id IN ('WL-008', '13221', '20215');
```

```sql
UPDATE trips_p2 as s
SET end_station_id = CAST(c.new_id AS varchar)
FROM id_changes_p2 as c
WHERE s.end_station_name = c.old_name;
```

*Name change*

- start_station

```sql
UPDATE trips_p2
SET start_station_name = CASE
  WHEN start_station_id = '57' THEN 'Clinton St & Roosevelt Rd'
  WHEN start_station_id = '61' THEN 'Wood St & Milwaukee Ave'
  WHEN start_station_id = '732' THEN 'Hegewisch Metra Station'
  END
WHERE start_station_id IN ('57', '61', '732');
```

```sql
UPDATE trips_p2 as s
SET start_station_name = c.new_name
FROM name_changes_p2 as c
WHERE s.start_station_name = c.old_name;
```

- end_station

```sql
UPDATE trips_p2
SET end_station_name = CASE
  WHEN end_station_id = '57' THEN 'Clinton St & Roosevelt Rd'
  WHEN end_station_id = '61' THEN 'Wood St & Milwaukee Ave'
  WHEN end_station_id = '732' THEN 'Hegewisch Metra Station'
  END
WHERE end_station_id IN ('57', '61', '732');
```

```sql
UPDATE trips_p2 as s
SET end_station_name = c.new_name
FROM name_changes_p2 as c
WHERE s.end_station_name = c.old_name;
```

> P.S. After cleaning both tables, some data are added into **id_changes_p1.csv** & **name_changes_p1.csv**. Worry not since all the uploaded and linked files are updated.

&nbsp;

<h3 align = "center" id = "combine-table-trips"><strong>Combine table: trips</strong></h3>

Before combining both tables, some changes has to be made first. By checking the schema of the tables we can see the differences in their data. After modification, merge both tables into one and save as **trips** table.

<h4 id = "schema"><strong>Schema</strong></h4>

- trips_p1

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


- trips_p2

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


**Table changes**:
- *trips_p1*
  - change data type of column **trip_id**: *bigint* to *varchar*.
  - rename column: **trip_id** to **ride_id**
- **trips_p2**
  - remove columns: **start_lat**, **start_lng**, **end_lat** & **end_lng**.
  - fill missing data: **trip_duration** by using start_time and end_time.
  - change data type of columns **start_station_id** & **end_station_id**: from *varchar* to *bigint*.

&nbsp;

<h4 id = "sql-query-2">SQL Query</h4>

```sql
CREATE TABLE trips AS
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
FROM trips_p1
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
FROM trips_p2;
```

<h3 align = "center" id = "null-values"><strong>NULL values</strong></h3>

Some data are omitted in table **trips_p2** that has NULL **station_names** & **station_id**. Although these records doesn't contain any data about its *station_names* and *station_IDs*, these have a coordinates which then can be used later on to identify the nearby **DIVVY** station name. The data will be filled in the future after finishing my studies on Webscraping in Python.

*Number of records*

| Data     | No. of records |
| -------- | -------------- |
| trips_p1 |     24,426,783 |       
| trips_p2 |      9,136,746 |
| trips    |     33,563,529 |
| NULL     |      1,158,190 |


*Percent of NULL values*

| Table    | NULL values |
| -------- | ----------- |
| trips_p1 |         0 % |
| trips_p2 |     12.68 % |
| trips    |      3.45 % |

Divvy-bikeshare dataset contains 3.45% of NULL values. Since the percentage is small, we can then omit these records and proceed with the analysis.

&nbsp;    

<h4 id = "sql-query-3">SQL Query</h4>

*Number of records*

```sql
-- trips_p1: 24,426,783
SELECT COUNT(*) FROM trips_p1;

-- trips_p2: 9,136,746
SELECT COUNT(*) FROM trips_p2;

-- total: 33,563,529
SELECT COUNT(*) FROM trips;

-- total NULL records: 1,158,190
SELECT CAST(COUNT(*) AS numeric) 
  FROM trips_p2
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
  FROM trips_p1
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM trips_p1)
			 , 2) AS NULL_values_percent

-- none
```

```sql
-- trips_p2
SELECT round(100 * 
 (SELECT CAST(COUNT(*) AS numeric) 
  FROM trips_p2
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM trips_p2)
			 , 2) AS NULL_values_percent

-- 12.68% of data is NULL
```

Since only table **trips_p2** has NULL values, we then use the table **trips_p2** as reference.

```sql
-- entire dataset
SELECT round(100 * 
 (SELECT CAST(COUNT(*) AS numeric) 
  FROM trips_p2
  WHERE (start_station_name IS NULL 
  AND start_station_id IS NULL)
  OR (end_station_name IS NULL
  AND end_station_id IS NULL)
  ) /	
 (SELECT CAST(COUNT(*) AS numeric)
  FROM trips)
			 , 2) AS omitted_data_percent

-- 3.45% of data is NULL
```

&nbsp;

<h2 align = "center" id = "stations-table">Stations table</h2>

<h3 id = "missing-stations"><strong>Missing stations</strong></h3>

Now that we have the missing stations from the 2 tables, trips_p1 & trips_p2, combine them into one table and save as [missing_stations.csv](https://github.com/ca-ros/divvy-bikeshare/blob/master/data%20wrangling/csv%20files/stations_cleaning/missing_stations.csv).

*Import csv file*

- missing_stations.csv

Add the missing stations to Stations table

```sql
-- Import
COPY stations (
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


<h3 id = "cleaning"><strong>Cleaning</strong></h3>

Some IDs need to change from the table.

*Import csv file*

- id_changes_stations.csv

```sql
-- Create table
CREATE TABLE id_changes_stations (
  name varchar,
  id bigint);
```

```sql
-- Import
COPY id_changes_stations (name, id)
FROM 'D:/Github/divvy-bikeshare/csv files/stations/id_changes_stations.csv'
DELIMITER ',' CSV HEADER;
```

*ID change*

```sql
UPDATE stations as s
SET id = c.new_id
FROM id_changes_stations as c
WHERE s.name = c.old_name;
```

<h2 id = "cleaned-dataset">🧹 Cleaned Dataset</h2>

Download here:
- [Kaggle](www.kaggle.com/dataset/e116a4d4f9c1900cf2b5b0b6a9270e20a378a4a18d209f5277253e8afbf2ef7d)
- [Google Drive](https://drive.google.com/file/d/1xhHuh9WXHtIBLPV6OO-a62th6Ev27jmM/view?usp=sharing)

> This dataset contains trip duration less than 60 seconds and stations with null values. I kept it incased someone need the entire data. Stations with null values will be filled with data in the future after I have a good grasp in **Machine Learning** and **Web Scraping**.
