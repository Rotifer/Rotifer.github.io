# Learning and using DuckDB with football data 🦆⚽

<div style="background-color:#E5E4D7">

<strong>Note for following along</strong>

<ul> 
    <li> Check out the  <a href="https://github.com/Rotifer/duckdb_epl">GitHub repo</a></li>
    <li> See the data in Google sheets format <a href="https://docs.google.com/spreadsheets/d/15EpbhgQibpv2haCeWsM77uApxgS5zYfq/edit?gid=1237416221#gid=1237416221">here.</a></li>
</ul>

</div>

I am starting a series of blogs where I use DuckDB to analyse data from the [EngLish Premier League](https://en.wikipedia.org/wiki/Premier_League). I have collected relevant data for season 1992-1993 to 2023-2024 for analysis in DuckDB. This is a small dataset but it is also one that lends itself to some interesting queries, interesting if you like football but also interesting from an SQL point of view. 

I firmly believe that a great way to learn SQL is to explore data that you are interested in either professionally or as a hobby. My hope is that the football data set we are going to explore is sufficiently interesting to a reasonably wide range of people and also simple enough for those with no knowledge of or interest in football to follow along.

## Objectives

- Explore advanced SQL features supported by DuckDB 🦆
- Learn how to use the DuckDB Command Line Interface (CLI) with bash scripts and general Unix tools like _sed_ and _awk_.
- Provide an analysis-ready dataset that others can use both with DuckDB and other database software such as SQLite, PostgreSQL, etc.

## Topics to be covered

- Cleaning and aggregating data using Unix tools in conjunction with DuckDB.
- Analysing the the data.
- Generating reports.
  

## Why football? ⚽?

- It is something I am interested in - that is a strong motivator and helps in formulating queries.
- It is easy to verify query results against what is already recorded in places like Wikipedia.
- Although the dataset is very small, it is composed of different types: dates, numbers and text.
- Missing data is always a complication and we have that too.

## Some football background

By football, I mean what is known as "soccer" in some English-speaking regions. The English Premier League (EPL) was formed in 1992 and is England's top tier football division for the men's game. It was initially composed of 22 clubs but this was reduced to 20 clubs in season after three seasons and has remained at this number ever since. 

A season begins in August and continues into May of the following year, so the season extends over two calendar years. Over that time each club plays every other club in the EPL twice: once at _home_ (in its own stadium) and once _away_ (in its opponent's club stadium). For example, each season, Liverpool plays Manchester United at its own stadium, Anfield, and at Manchester United's stadium, Old Trafford. 

When two clubs play each other in a match, three outcomes are possible:

1. A draw: Neither team scored or they scored the same number of goals.
2. A home win: The club playing at home scored more goals than the opponent away club.
3. An away win: The club playing away from home scored more goals than the opponent home club.

Depending on the outcome, points are awarded to the clubs involved in a given match. The winner gets three points but if the game is drawn each club is awarded one point.

Here is an example that should clarify the various terms I have used above:

In season 1999_2000 Newcastle United played Southampton at home (St James Park in Newcastle) on the date 2000-01-16 and beat them 5 goals to nil (zero). Therefore, for that match, Newcastle were awarded 3 points and Southampton received zero points.

All clubs start the season with zero points and they accumulate them over a season depending on their results against other clubs. For the current 20 club in the EPL, each club plays 38 games so the theoretical maximum a club can receive over the season is 114 points (3 X 38), that is if they win __all__ their matches (not likely and has never happened). The club with the most points at the end of the season wins the EPL and is declared "Champions of England". If clubs are tied on points, _goal difference_ comes in to play. It is the club's total number of goals scored minus the number of goals conceded in all league league matches over that season. This, as we will see, has actually happened in one season.

The three clubs with the least points are relegated to the second tier of English football called _The Championship_ and are replaced by three promoted clubs from that tier. Once again, for relegation and promotion, rankings for clubs tied on points is decided by goal difference. This continuous relegation-promotion cycle means that the EPL clubs differ over the seasons which is something else we will explore with queries.

## Tools and resources

- I use the DuckDB CLI and the excellent [Harlequin IDE](https://harlequin.sh/)
- All the input and output data are available at this Git repo: [duckdb_epl](https://github.com/Rotifer/duckdb_epl)
- You can use [SQL Workbech](https://sql-workbench.com/) in your blowser to explore the DuckDB database file _epl.ddb_ in the repo directory _db_files_
- The final tables in this database are very small so I have made them available in a Google Sheets spreadsheet, [link](https://docs.google.com/spreadsheets/d/15EpbhgQibpv2haCeWsM77uApxgS5zYfq/edit?usp=sharing&ouid=102009104893541016477&rtpof=true&sd=true). The sheet names correspond to the DuckDB table names (in schema _main_) except that for views, I have dropped the "vw_" prefix.
- The is also an Excel version in the output_data directory of the repo, the file is named epl_YYYYMMDD.xlsx where YYYYMMDD represents the date the file was generated. Any edits or fixes will be present in this file and in the DuckDB database itself.

## Next up

The next entry will include a fairly lengthy bash script that aggregates the data and prepares it for upload into DuckDB. A few more data munging blog posts will follow but once we have clean table data, we can start to dig in to DuckDB SQL to answer some interesting questions.


