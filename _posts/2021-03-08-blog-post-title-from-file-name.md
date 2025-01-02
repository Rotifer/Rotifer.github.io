## Blog Post Title From First Header

Blog created using the instructions kindly provided by Chad Baldwin
[Building a Free Blog with GitHub Pages in Minutes](https://chadbaldwin.net/2021/03/14/how-to-build-a-sql-blog.html)


---

### Testing DuckDB SQL

```sql
CREATE TABLE clubs(
  club_code VARCHAR PRIMARY KEY,
  club_name VARCHAR,
  club_given_name VARCHAR);
```

### Some bash code

```sh
#!/usr/bin/bash

# Declare variables.
OUTPUT_DIR="../output_data/"
INPUT_DIR="../source_data/"

# Remove the final output file if it already exists.
if [[ -f ${OUTPUT_DIR}seasons_1993_2023.tsv ]]; then
  rm ${OUTPUT_DIR}seasons_1993_2023.tsv 
fi
```
