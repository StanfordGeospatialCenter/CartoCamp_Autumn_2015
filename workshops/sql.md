## Introduction to SQL and PostGIS in CartoDB

Andy Eschbacher, Map Scientist, CartoDB

### Find this document here: [http://bit.ly/cdb-stanford-sql](http://bit.ly/cdb-stanford-sql)

### Roughly the Structure of CartoDB

![Structure of CartoDB](http://i.imgur.com/9XAYbtZ.png "Structure of CartoDB")

### What is SQL?

Structured Query Language --> "ask questions of data"

#### Schema

Each dataset in CartoDB has a formal structure of columns with a specific data type and name, and rows that are uniquely identified by a column number `cartodb_id`.

#### SQL

SQL is deceptively simple. The common part of the language is just a handful of keywords: SELECT, FROM, WHERE. You can do maybe 75% of what you'll ever need to do with these keywords. Unlike other programming languages, SQL's complexity ramps up quickly.

A typical SQL statement asks for column names from a table given a constraint on the values in the columns.

For instance, we could ask: Give me all the names, heights, and favorite cookies of people in this room whose shoe size is between 10 and 11 and whose father's name is Peter.

The query would look like this:

```sql
SELECT
    name,
    height, 
    favorite_cookie
FROM
    everyone_in_this_room_table
WHERE
    shoe_size > 10 
   AND
    shoe_size < 11 
   AND 
    father_name = 'Peter'
```

This would give back the following result (probably):

```text
|  name  |  height  |  favorite_cookie  |
-----------------------------------------
|  Andy  |   1.78   |      Pecan        |
```

#### Which SQL does CartoDB use?

CartoDB uses [PostgreSQL](http://www.postgresql.org/) because it allows for extensions. One extension specifically allowed for CartoDB to exist: [PostGIS](http://postgis.net). This allows one to do geospatial analysis in a database framework. All queries that don't require admin-level access can be run in your CartoDB database.

PostGIS allows you to do queries like this:

```sql
SELECT superhero.name
FROM city, superhero
WHERE ST_Contains(city.geom, superhero.geom)
AND city.name = 'Gotham';
```

The key piece here is the `ST_Contains()` portion: it gives back true or false depending on whether the superhero is inside of the city. This can be handy when you have two datasets and want to count how many points from one dataset occur in a polygon from another dataset.

### Getting help with SQL in CartoDB

If it's not spatial search for: 'postgresql ...' where the ... is what you need to know. For instance, 'postgresql string operators'.

If the data is spatial, search for: 'postgis ...'. For instance, 'postgis area'.

There is also [GIS StackExchange/CartoDB](http://gis.stackexchange.com/questions/tagged/cartodb) has ~800 questions and answers. [GIS SE/PostGIS](http://gis.stackexchange.com/questions/tagged/postgis) has many more.

### Our datasets

We'll be working from Stace Maples' dataset on clowns. Retrieve it by copying the following link and importing it into your account:

```html
http://eschbacher.cartodb.com/api/v2/sql?q=SELECT%20*%20FROM%20clowns_of_america_international_member_dataset_anon&format=shp&filename=clowns_sql
```

Notice that this is a SQL API call: we're specifying the custom SQL with the `q` parameter, the download format with the `format` param, and the filename with `filename` param. We could do any arbitrary SQL query here, even against multiple tables at once.

### WHERE to go next

We can discover a lot of the comparison operators by using the filters. The comparison operators evalute to `true` or `false` statements. We could chain multiple logically statements together with the AND and OR operators. For instance, a filter like this:

```sql
SELECT *
FROM cookie_preferences
WHERE (age > 13 OR age < 7) AND (favorite_cookie = 'Chocolate Chip'`)
```

against a table like this:

```text
|  name  |  age  |  favorite_cookie  |
--------------------------------------
| Andy   |  33   |   Pecan           |
| Linden |  13   |   Chocolate Chip  |
| Xiu    |  12   |   Chocolate Chip  |
| Otto   |  49   |   Peanut Butter   |
| Lisa   |  6    |   Chocolate Chip  |
```

will produce the following:


```text
|  name  |  age  |  favorite_cookie  |
--------------------------------------
| Lisa   |  6    |   Chocolate Chip  |
```

Notice that the filter conditions are working on numbers and strings. More information [here](http://www.postgresql.org/docs/9.3/static/functions-comparison.html) on these comparison operators. Read more about the [logical operators](http://www.postgresql.org/docs/9.3/static/functions-logical.html).

You can also have conditions on sets:

```sql
SELECT * 
FROM awesome_table
WHERE name IN ('Stace','Kim','David','Bill')
```

You can also use `NOT IN` for negation.

And test whether values are null or not:

```sql
SELECT *
FROM cat_memes
WHERE (cat_name IS null) And (youtube_url IS NOT null)
```

Two gems that don't show up in the filters are: LIKE and ILIKE. They match strings that are similar to another string, like this:

```text
'abc' LIKE 'abc'    true
'abc' LIKE 'a%'     true
'Abc' ILIKE 'a%'    true
'abc' LIKE '_b_'    true
'abc' LIKE 'c'      false
```
`ILIKE` is especially good for messy data where capitalization is not standardized. You can read much more about [pattern matching in PostgreSQL](http://www.postgresql.org/docs/9.3/static/functions-matching.html).

### Challenge(s) \#1

1. Find all clowns who have more than one name.
1. Find all clowns who have more than one word in their name (that is, there is a space), and are in a zip code +/- 100 from Stanford's (94305).
1. Find all clowns that have an empty `country` field OR where `the_geom` is null-valued.
1. Find all clowns who are not in the United States and whose country is not an empty string.


**Before moving on**, let's reset our map to the default SQL (`SELECT * FROM clown_table`), which you can get by clicking 'clear view' at the bottom of the SQL tray if you have a custom SQL statement applied.

Let's style our map as follows: 

```css
#clowns{
  marker-fill-opacity: 0.9;
  marker-line-color: #FFF;
  marker-line-width: 0;
  marker-line-opacity: 1;
  marker-placement: point;
  marker-type: ellipse;
  marker-width: 10;
  marker-fill: #0F3B82;
  marker-allow-overlap: true;
  marker-comp-op: plus;
}
```

The composite operation `plus` adds the individual rgb channels of the markers to show something like an intensity map of the markers at locations.

Let's also configure our infowindows or hovers to show all the information associated with a click or hover event.

### Aggregate functions

We have a problem with our data visualization. There are tons of clowns in the same zipcode. Since the zipcode centers are the rough locations of our clowns, they are stacked on top of each other, so our click events can't find all of the clowns.

![now showing all clowns](http://i.imgur.com/f9kfkiN.png)

A good strategy here is to group all of the clowns who share the same physical location. Luckily, we have a SQL keyword (or two) for that: `GROUP BY`.

`GROUP BY` partitions your data into discrete groups, allowing you to perform functions on a group-by-group basis. For instance, if we want to find the average age within a zipcode (if we were so lucky as to have that data here).

Looking at the [PostgreSQL aggregate function](http://www.postgresql.org/docs/9.3/static/functions-aggregate.html) page, we have several options for aggregating the date in our table.

Our goal is to display all of the names of clowns at a single location or zip code.

We can recast our datatable by aggregating the clowns that are at the same location and city. Within those groups, let's:

1. Build up a list of clown names (using `string_agg`)
1. Build up a list of zip codes (using `string_agg`)
1. Count the number of clowns residing there (using `count(1)`)
1. Calculate the square root of the count for visualizing the number of clowns
1. Grab one of the `cartodb_id`s so we can enable click events


```sql
SELECT 
  the_geom_webmercator,
  city,
  string_agg(clown_name,', ') clown_names, 
  string_agg(zipcode::text,', ') zips,
  count(1) cnt,
  sqrt(count(1)) sqrtcnt,
  max(cartodb_id) cartodb_id
FROM 
  clowns_sql
GROUP BY
  the_geom_webmercator, city
```

Click on a symbol and select all of the columns to be displayed in it, like this:

![SQL Tray](http://i.imgur.com/wgI5NsS.png "SQL Tray")

Next, let's configure the style to more effectively visualize the clowns in an area. We will create markers sized by the square root of the number of clowns in an area. We can do this with CartoCSS as follows:

![sized by number of clowns](http://i.imgur.com/lJDUTbN.png "sized by number of clowns")

```css
#clowns{
  marker-fill-opacity: 0.9;
  marker-line-color: #FFF;
  marker-line-width: 0;
  marker-line-opacity: 1;
  marker-placement: point;
  marker-type: ellipse;
  marker-width: 5*[sqrtcnt];
  marker-fill: #0F3B82;
  marker-allow-overlap: true;
  marker-comp-op: plus;
  [zoom > 5] {
  	marker-width: 10*[sqrtcnt];
  }
}
```

Looking through our results, this is a much better map, but there are some empty strings and uniform capitalization. To make it more legible we could apply some string operations.

### String operations

Now let's do a little more string cleaning.

For reference, PostgreSQL string operations are listed in the documentation here:
[http://www.postgresql.org/docs/9.3/static/functions-string.html](http://www.postgresql.org/docs/9.3/static/functions-string.html).

To handle the conditional nature of empty string versus not, we can use `CASE` statements ([docs on them](http://www.postgresql.org/docs/9.3/static/functions-conditional.html)). These are almost identical to `IF THEN` statements:

```sql
SELECT 
    CASE is_scary
    WHEN true
      THEN clown_name || ' should not go to kids parties'
    ELSE name || ' can go to parties'
    END As go_to_party_or_not
FROM supplemental_clown_table
```

The `||` operators add strings together.

If our table looks like this:

```text
|      name       |  is_scary  |
--------------------------------
| SNICKERS        |   true     |
| BISCUIT D CLOWN |   true     |
| FLOOZIE DOO     |   false    |
| PERFLESS        |   true     |
| DAFFY-DILLY     |   true     |
```

The SQL command above would produce:

```text

|             go_to_party_or_not                |
-------------------------------------------------
| SNICKERS should not go to kids parties        |
| BISCUIT D CLOWN should not go to kids parties |
| FLOOZIE DOO can go to parties                 |
| PERFLESS should not go to kids parties        |
| DAFFY-DILLY should not go to kids parties     |
```

We can use these `CASE` statements inside of our `string_agg` function to process our empty strings (`''`) into `UNKNOWN` instead. We can also use the `initcap` function to only make the initial character of each word capitalized. Notice that we are compositing our operations more and more.

We can also use the keyword `DISTINCT` to remove duplicates from our zipcode list.


```sql
SELECT 
  string_agg(
    initcap(
      CASE clown_name 
      WHEN '' 
      THEN 'UNKNOWN' 
      ELSE clown_name 
      END
    ),
    ', '
  ) clown_names, 
  the_geom_webmercator,
  count(1) cnt,
  sqrt(count(1)) sqrtcnt,
  string_agg(DISTINCT zipcode::text,', ') zips,
  max(cartodb_id) cartodb_id
FROM 
  clowns_sql
GROUP BY
  city, the_geom_webmercator
```

See working map [here](https://team.cartodb.com/u/eschbacher/viz/2e12c958-6731-11e5-a90c-0e9d9d43ad5d/public_map).

### Challenge \#2

1. For clowns with two names listed, choose only the first name in the list. Check the [string functions](http://www.postgresql.org/docs/9.3/static/functions-string.html) for help
2. Calculate the average length of a clown's name for each location.
3. Show only the top 20 locations for clowns

### JOINing datasets

Next, let's find out the number of clowns in each state. To do that we need to get some state polygons.

Let's go back to our dashboard, change to data view, and then hit 'New Dataset'. Search for 'states' and look for the dataset titled 'USA states'. Click on it once to import it into your account.


Now that we have some polygons to work with, we can find out how many points (clowns) intersect with each polygon (state). We can use the PostGIS functions, [`ST_Contains`](http://postgis.net/docs/ST_Contains.html) or [`ST_Intersects`](http://postgis.net/docs/ST_Intersects.html).

```sql
SELECT
  count(c.*) As cnt,
  ST_Transform(s.the_geom,3857) As the_geom_webmercator
FROM 
  ne_50m_admin_1_states As s
JOIN
  clowns_sql As c
ON ST_Contains(s.the_geom, c.the_geom)
GROUP BY s.the_geom
```
This is all well and good, but it may reflect more of a population map than a map showing the signifance of a state's clown representation.

If our state dataset had population for each state, we could normalize by population. We don't have that data here, so we could opt instead to normalize by the area of the state, which we can calculate with PostGIS's [`ST_Area`](http://postgis.net/docs/ST_Area.html). 

We're specifically interested in units of square meters, which we can obtain by casting our geometry to ['geography'](http://postgis.net/docs/using_postgis_dbmanagement.html#PostGIS_Geography)... where some magic happens. All calculations are done on a sphere instead of on a plane.

We can get the number of clowns per square meter, but that turns out to be a very small number. Instead, we can get the number of clowns per 1000 square kilometer in a state. Since 1 km = 10^3 m, 1 km^2 = 10^6 m^2. Therefore, 1000 km^2 = 10^9 m^2.

Our query then looks like:

```sql
SELECT
  count(c.*) As cnt,
  1e9 * count(c.*) / ST_Area(s.the_geom::geography) As cnt_p_area,
  ST_Transform(s.the_geom,3857) As the_geom_webmercator
FROM 
  ne_50m_admin_1_states As s
JOIN
  clowns_sql As c
ON ST_Contains(s.the_geom, c.the_geom)
GROUP BY s.the_geom
```

Challenge \#3

_Find the average distance every clown is from the clown's state center. Hint: use ST_Distance(geom1,geom2) and ST_Centroid(geom)_

## Resources

+ CartoDB's free Map Academy: http://academy.cartodb.com/courses/04-sql-postgis.html
+ Codecademy's SQL course: https://www.codecademy.com/courses/learn-sql
+ Boundless's PostGIS tutorials: http://workshops.boundlessgeo.com/postgis-intro/
