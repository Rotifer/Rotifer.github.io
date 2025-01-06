# Creation of the _clubs_ and _matches_  tables 🦆⚽

We now have our data in two tables in the _staging_ scehma but it is not yet analysis ready. Just to recap, our raw data table names are:

1. _season_1992_1993_raw_
1. _seasons_1993_2023_raw_


First, we are also going to create and populate another table in schema _main_ called _clubs_ to map club names to commonly used three letter club codes. The data for this small look-up table was pre-prepared in Google Sheets 

The clean up we will need to do on our two raw to tables to get to analysis-ready data is required for most data tasks; we are rarely given analysis-ready data and if we are, it usually means someone, somewhere put in considerable effort to clean the data for us. The tasks we will see here also showcase DuckDB's excellent data wrangling capabilities; it really is a first class tool for these types of task and I will highlight its impressive strengths in this regard as we transform our raw data into clean, analysis-ready tables.


## Table _clubs_

### Table creation

We create the table in schema _main_ using the following SQL

```tsql
USE main; -- Ensure this table is created in schema main
CREATE OR REPLACE TABLE clubs(
  club_code VARCHAR PRIMARY KEY,
  club_name VARCHAR,
  club_given_name VARCHAR);
```

I have added a primary key, the club code, to ensure that the club code is unique. We are not building a transactional database (OLTP) but rather an analytical one so I will not spend much time discussing database keys or database design but I will add primary and foreign keys if it makes sense to do so.

__Note__: The "OR REPLACE" is a non-standard DuckDB addition to SQL. It is standard SQL for view creation but in SQLite or  PostgreSQL we would need a "DROP TABLE IF EXISTS <table_name>;" to get the same effect. DuckDB supports this syntax too so we could use it if compatibility was a concern.

### Commenting a table and its columns

A nice feature of DuckDB is the ability to add comments to objects that are stored in its system catalogs. The syntax is identical to PostgreSQL and we will add comments to all our tables, their columns, views or macros as we proceed.

```tsql
COMMENT ON TABLE clubs IS 'A lookup that maps club three-letter codes to names given in source files and the official club name. Records every club that has played in at least one EPL season.';
COMMENT ON COLUMN clubs.club_code IS 'A recognised uppercase three letter code for clubs to be used in other tables.';
COMMENT ON COLUMN clubs.club_name IS 'The official name of the club.';
COMMENT ON COLUMN clubs.club_given_name IS 'The club name used in source files which may or may not be the same as the official club name.';
```
### Populating the table

The file which was curated in Google Sheets and then exported as a TSV is uploaded into our new, heavily commented table as follows:

```tsql
INSERT INTO clubs(club_code,
                  club_name, 
                  club_given_name) 
SELECT 
  club_code, 
  club_name, 
  given_name 
FROM '../source_data/clubs.tsv';
```

__Note__: The three letter codes I have used here are those given Wikipedia. I am not sure if they are official but they are suitable identifiers because they are both short and unambiguous while club names are tedious to type and ambiguous; is it "Manchester United", "Man U", or just "United"? The three letter code "MUN" is much easier. When we want to use the full club name, in reports for example, we can join to this table to get it.



## Create the _matches_ table

This is our main table and it records details for all EPL matches played from season 1992_1993 to season 2023_2024. We will create the table in schema _main_ and then populate it with the data we validate and format from the raw tables in schema _staging_

```tsql
USE main;
CREATE SEQUENCE seq_match_id START 1;
CREATE OR REPLACE TABLE matches(
  mid INTEGER PRIMARY KEY DEFAULT NEXTVAL('seq_match_id'),
  season TEXT NOT NULL,
  mdate DATE,
  mtime TIME,
  hcc TEXT NOT NULL,
  acc TEXT NOT NULL,
  hcg TINYINT NOT NULL,
  acg TINYINT NOT NULL
);
```
### Use of sequences

We have used the _sequence_ object to create a monotonically increasing number to use as a primary key. while not strictly necessary, it is useful to allow us to easily identify individual matches and the ID will be generated automatically when we insert rows later.

### Document our table

We will add comments as we did for table _clubs_.

```tsql
COMMENT ON TABLE matches IS 'A record of clubs, goals scored/conceded with time and date details (if available) for every match from season 1992_1993 to 2023_2024.';
COMMENT ON COLUMN matches.mid IS 'The match ID: Primary key autogenerated from a sequence.';
COMMENT ON COLUMN matches.season IS 'The season the match was played in.';
COMMENT ON COLUMN matches.mdate IS 'The date on which the match was played, if available.';
COMMENT ON COLUMN matches.mtime IS 'The time the match was played. Data only available for recent seasons.';
COMMENT ON COLUMN matches.hcc IS 'Home club code: The club playing at its home stadium/';
COMMENT ON COLUMN matches.acc IS 'Away club code: The club playing away from its home stadium';
COMMENT ON COLUMN matches.hcg IS 'Home club goals: The number of goals scored by the home club.';
COMMENT ON COLUMN matches.acg IS 'Away club goals: The number of goals scored by the away club.';
```
__Note__: I have deliberately used short column names for the table but I have fully spelled out the columns' purpose in the comments where the abbreviated names are explained; "acc" means "Away Club Code", for example. This approach makes query writing simpler and we can avoid using aliases.

## Insert data into the table _matches_

Our _matches_ table is now ready to receive data but before we insert it, we need to check it and clean it up.

### Season 1992-1993

The data giving match scores for season 1992-1993 was downloaded from Wikipedia as discussed in the [previous post](https://rotifer.github.io/2025/01/04/loading-and-viewing-data-in-duckdb.html). It is in a pivot format that uses row names to represent the home club and column names to represent the away club with the intersection cell giving the score as home club goals scored followed by a dash of some sort followed by the away club goals scored.


Question: What is the dash-like character separating the score in the data cells?

VS Code says: "The character U+2013 "–" could be confused with the ASCII character U+002d "-", which is more common in source code"

[What is _endash_](https://stackoverflow.com/questions/59839683/en-dash-giving-different-ascii-values)

Our final objective is to create a version of the data for this season which we can integrate with the data for all other seasons. We will do this step-wise and create intermediate tables at each step. Each intermediate table will be the input for the subsequent step. Once we have achieved our goal, we will remove the intermediate tables. Note, all of this activity takes place in the _staging_ schema.

#### Un-pivoting

We need to "unpivot" the data, that is turn the "wide and short" representation into "long and thin". Hadley Wickham of R fame describes the long thin format as ["tidy data"](https://vita.had.co.nz/papers/tidy-data.pdf)
The DuckDB documentation on unpivoting is excellent: [UNPIVOT Statement](https://duckdb.org/docs/sql/statements/unpivot.html). It is easy to follow and gives examples you can try out yourself.

```tsql
USE staging;
CREATE OR REPLACE TABLE season_1992_1993_unpivot AS
UNPIVOT season_1992_1993_raw
ON COLUMNS(* EXCLUDE "Home \ Away")
INTO
  NAME away_club_code
  VALUE score;
```



#### Beaking the scores into home and away columns

The scores are separated by that odd _endash_ character and we need to break them appart. The _STRING_SPLIT_ function is one of the many very useful DuckDB string functions and it takes the string and the split character (CHR(8211) represents endash) to return an array. We extract the two elements from this array  using the array element access notation []. DuckDB arrays are 1-based so [1] returns the home club goals while [2] returns the away club goals.

```tsql
CREATE OR REPLACE TABLE season_1992_1993_scores AS
SELECT
  "Home \ Away" AS home_club_name,
  away_club_code,
  STRING_SPLIT(score, CHR(8211))[1] home_club_score,
  STRING_SPLIT(score, CHR(8211))[2] away_club_score
FROM season_1992_1993_unpivot;
```

#### Remove self v self rows

The DELETE relies on the fact that rows where the home and away club is the same the second element returned by the _STRING_SPLIT_ function will be _NULL_.

Check the count before the delete:

```tsql
SELECT COUNT(*)
FROM season_1992_1993_scores; -- -> 484
```

```tsql
DELETE FROM season_1992_1993_scores
WHERE away_club_score IS NULL;
```

```tsql
SELECT COUNT(*)
FROM season_1992_1993_scores; -- -> 462
```


#### CAST scores to integer type

Our scores are stored as strings (column type VARCHAR) but we want to convert them (CAST) to integers.

```tsql
CREATE OR REPLACE TABLE season_1992_1993_int_scores
AS
SELECT
  home_club_name,
  away_club_code,
  CAST(home_club_score AS TINYINT) home_club_score,
  CAST(away_club_score AS TINYINT) away_club_score
FROM season_1992_1993_scores;
```

Note, I have used TINYINT as the column type for scores because these is no need to use a larger integer type.


#### Join tables to convert club name to club code

Our data is almost ready to be loaded into our table _matches_ in schema _main_ but we need to do one more thing: convert the home club name to the home club code (_hcc_ in table _matches_). We can achieve this by joining to our _clubs_ table

```tsql
CREATE OR REPLACE TABLE season_1992_ready AS
SELECT
  '1992_1993' season,
  NULL mdate,
  NULL mtime,
  c.club_code hcc,
  sis.away_club_code acc,
  sis.home_club_score hcg,
  sis.away_club_score acg
FROM
  season_1992_1993_int_scores sis
  JOIN main.clubs c
    ON sis.home_club_name = c.club_name;
```

This table's data is now ready to be inserted into _matches_. The join replaces the club name with the club code and we have added three columns:

1. _season_: We know the rows relate to season _'1992_1993'_ so we repeat this string for every row.
2. We do not have the match dates so we just repeat NULL for each row and name the column _mdate_.
3. Similary for _mtime_, we repeat NULLS for this column also.

Filling in these columns isn't strictly necessary but I did it to emphasize how we can explicitly highlight NULLs for columns and to make the table the same structure as _matches_.

#### Append the data to table _matches_

A simple append query works here. Note, this query is executed from schema _main_ so we have to qualify the source table name with its schema in the FROM clause.

```tsql
USE main;
INSERT INTO matches(season,
                    mdate,
                    mtime,
                    hcc,
                    acc,
                    hcg,
                    acg)
SELECT
  season,
  mdate,
  mtime,
  hcc,
  acc,
  hcg,
  acg
FROM
  staging.season_1992_ready;
```

The sequence column (_mid_) assigns unique monotonically ascending numbers to all the new rows automatically.

👏👏 We now have data for our first season in the table _matches_. You can run a `SELECT COUNT(*) FROM matches`` to verify insertion of 462 rows. Our table is analysis-ready but unfortunately we don't have the match dates so we will need to be aware of this fact when we run queries for all seasons. Missing data is common and needs to be taken into account in analyses.

Clean up: We can `DROP` our intermediate tables now:

```tsql
USE staging;
DROP TABLE season_1992_1993_unpivot;
DROP TABLE season_1992_1993_scores;
DROP TABLE season_1992_1993_int_scores;
DROP TABLE season_1992_ready;
```

## Seasons 1993_1994 to 2023_2024

All the other seasons' data is available in the _staging table _epl_matches_1992_2024_. I spent some time verifying the data in this table and identified a number of issues we need to address:

__Issue 1__: There are some spurious rows in the table that contain no data except season. This arose because some of the downloaded source files had empty lines. You can see these empty lines by executing the following SQL:

```tsql
SELECT COUNT(*) 
FROM seasons_1993_2023_raw 
WHERE home_club_name IS NULL;
```

There are 697 such lines and they need to be filtered out.

__Issue 2__: The _match_date_ format is inconsistent, some dates have two-digit years and others have four-digit years. We can identify each group using regular expressions like so:

```tsql
-- Two-digit dates.
SELECT match_date 
FROM seasons_1993_2023_raw 
WHERE REGEXP_MATCHES(match_date, '\d{2}\/\d{2}\/\d{2}$');
-- Four-digit dates.
SELECT match_date 
FROM seasons_1993_2023_raw 
WHERE REGEXP_MATCHES(match_date, '\d{2}\/\d{2}\/\d{4}$');
```

When we do the date conversions, we have to account for these two different formats.

__Issue 3__: Our raw data uses the club names given by the file source website, but we want to map these to the three-letter club codes we assigned in our _clubs_ table. We will use the _clubs_ table to substitute the club names for club codes.

__Issue 4__: For missing _match_time_  values we have the "NA" (Not Available) flag, we need to convert these to NULLs and all valid date strings to _TIME_ type.

The following SQL effects all these transformations and creates a data output that we can append to the season 1992_1993 rows already in our _matches_ table.

```tsql
CREATE OR REPLACE TABLE staging.epl_matches_1992_2024 AS
SELECT
  season,
  CASE
    WHEN REGEXP_MATCHES(match_date, 
            '\d{2}\/\d{2}\/\d{2}$') THEN
      STRPTIME(match_date, '%d/%m/%y') 
    WHEN REGEXP_MATCHES(match_date,
            '\d{2}\/\d{2}\/\d{4}$') THEN
      STRPTIME(match_date, '%d/%m/%Y')
  END mdate,
  CASE
    WHEN match_time = 'NA' THEN NULL
    ELSE CAST(match_time AS TIME)
  END mtime,
  (SELECT club_code 
      FROM main.clubs 
    WHERE club_given_name = raw.home_club_name 
    LIMIT 1) hcc,
  (SELECT club_code 
      FROM main.clubs 
    WHERE club_given_name = raw.away_club_name 
    LIMIT 1) acc,
  home_club_goals hcg,
  away_club_goals acg
FROM seasons_1993_2023_raw raw
WHERE home_club_name IS NOT NULL;
```

The above SQL creates a new table whose data is ready to be appended to our _matches_ table.
As this is quite a large piece of SQL code, it merits some explanation.

- The first CASE statement uses a regex to determine the date format and then uses the _STRPTIME_ function to perform the type conversion (VARCHAR to DATE) using the appropriate format.

- The second case statement converts the _match_time_ string value to a _TIME_ type or to NULL if the string value is "NA".

- The two nested scalar SELECT statements in the main SELECT clause perform lookups and substitute the club name with the three letter club code.

- The WHERE clause filters out all spurious rows that arose from blank lines in the downloaded source files.

Our table can now be appended to the _matches_ table.

### Append to _matches_ table

```tsql
USE main;
INSERT INTO matches(season,
                    mdate,
                    mtime,
                    hcc,
                    acc,
                    hcg,
                    acg)
SELECT
  season,
  mdate,
  mtime,
  hcc,
  acc,
  hcg,
  acg
FROM
  staging.epl_matches_1992_2024;
```

👏👏  We now have all our data in _matches_, 12,406 rows, and can now start working on the data in earnest. The data is clean and the columns are all the correct data type.

## Summing up

- We now have our matches data in a a clean and analysis-ready format.
- DuckDB has many features which facilitate data cleaning and re-formatting:
  - Regular expressions
  - Arrays and lists
  - A comprehensive set of functions for manipulating all kinds of data
- Adding comments to tables and columns makes our database self-documenting

## Next up

We will use SQL to create a final table derived from the _matches_ table which will summarise the end-of-season rankings of each club hus giving us the winners and losers of each league from 1992-1993 to 2013-2024. This table's creation will introduce more DuckDB SQL. We can then dive deep into our data and use DuckDB to answer all kinds of interesting questions.