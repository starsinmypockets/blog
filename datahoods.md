Datahoods, exploring neighborhood-level data 
Part 1
Neighborhoods in GeoJSON and postGIS

If you live in Philly you know that you can't mention the name of a neighborhood in polite conversation without starting an argument about what the neighborhood is really called, where it starts and ends, who calls it what and why. The is a fascinating topic, and a conversation that I welcome, and would love to explore in greater depth in another blog post. For the moment, we will accept the neighborhood definitions as provided, with a grain of salt.

There is a lot of cool geojson out there. I wanted to explore 311 requests and other open data in the context of Philadelphia's neighborhoods so I started looking around for the data that I needed and found some interesting resources along the way:

https://www.opendataphilly.org/ - The official open data repository for the city.
https://github.com/blackmad/neighborhoods - A giant repository of open source geojson files

We'll use the following file for our tutorial: https://github.com/blackmad/neighborhoods/blob/master/philadelphia.geojson 

We can use the geoJSON as-is in a front-end tool like Leaflet[LINK], which is what we did[LINK].

We also want to get the geojson into a database so that we can perform queries at the neighborhood level. Here's how we did it.

We'll assume that you have postgres running in debian or a similar environment[LINK], and that you have postgis installed[LINK]. We also use a node-js tool to import the geoJSON.

Open up the shell and create a new database (note that we are showing the shell responses below the commands which follow the `=#` prompt):
```
postgres=# create database datahoods;
CREATE DATABASE

postgres=# \c datahoods 
psql (10.1, server 9.6.6)
You are now connected to database "datahoods" as user "postgres".
```

Enable the postgis extension:
```
datahoods=# CREATE EXTENSION postgis;
CREATE EXTENSION
```

Create a table to store the raw geojson:
```
datahoods=# CREATE TABLE geodata (id integer PRIMARY KEY, data json);
CREATE TABLE
```
From the command line install postgres-import-json[LINK] and import the geojson file to your postgres database:
`$> npm install -g postgres-import-json`
`$> postgres-import-json -f Neighborhoods_Philadelphia.json -d datahoods -t geodata -h localhost -p 5432 -u postgres`

Postgres has support for json (and jsonb not covered here), but I would prefer to store values as postgres/postgis native types, and avoid json operations in my subsequent queries. To this end, we will process the json into a neighborhood table:


Now with the data in postgres let's create a table where we can store our neighborhood records as native postgr"
```
datahoods=# CREATE TABLE neighborhoods (name text PRIMARY KEY, the_geometry geometry, listname text, mapname text, shape_leng numeric, shape_area numeric);
CREATE TABLE
```

Lets drill down through the json to extract our neighborhood rows. We'll use a couple steps to help clarify what is happening.
```
datahoods-# WITH features AS (
datahoods(#  SELECT json_array_elements(json_extract_path(data, 'features')) AS feature FROM geodata
datahoods(# ) SELECT 
datahoods-#  feature->'properties'->>'name' AS name,
datahoods-#  feature->'properties'->>'listname' AS listname,
datahoods-#  feature->'properties'->>'shape_leng' AS shape_leng,
datahoods-#  feature->'properties'->>'shape_area' AS shape_area
datahoods-#   FROM features;
```

We're on the right track. This returns the non-geometry data we will need to build our leaflet app:

```
         name         |          listname           |  shape_leng   |  shape_area   
----------------------+-----------------------------+---------------+---------------
 PENNYPACK_PARK       | Pennypack Park              | 87084.2855886 | 60140755.7554
 OVERBROOK            | Overbrook                   | 57004.9246069 | 76924994.536
 GERMANTOWN_SOUTHWEST | Germantown, Southwest       | 14880.7436082 | 14418666.0223
 EAST_PARKSIDE        | East Parkside               | 10885.7815353 | 4230999.8669
 GERMANY_HILL         | Germany Hill                | 13041.9390872 | 6949968.42772
 MOUNT_AIRY_EAST      | Mount Airy, East            | 28845.549808  | 43152469.5661
 MECHANICSVILLE       | Mechanicsville              | 5941.33261823 | 1646440.4726
 DEARNLEY_PARK        | Dearnley Park               | 17330.8420224 | 16215770.7346
 WISSAHICKON_HILLS    | Wissahickon Hills           | 7113.02864233 | 3048525.57286
 WISSINOMING          | Wissinoming                 | 29226.4744992 | 42361155.0398
 BELLA_VISTA          | Bella Vista                 | 8360.05369559 | 4253834.71172
```

Now we will use PostGIS to extract the geometry from the geoJSON.

```
datahoods=# WITH features AS (
datahoods(#  SELECT json_array_elements(json_extract_path(data, 'features')) AS feature FROM geodata
datahoods(# )
datahoods-# SELECT ST_SetSRID(ST_GeomFromGeoJSON(feature->>'geometry'), 4326)
datahoods-#   FROM features;
```

As in the previous step, we return an array of geoJSON features. Then we use three PostGIS functions to convert the geoJSON feature object into a geometry object. Let's put it all together and create our neighborhoods table with geojson and metadata:

Now lets clean it up a little and put it all together to populate our neighborhoods table:
```
datahoods=# WITH features AS (
datahoods(#  SELECT json_array_elements(json_extract_path(data, 'features')) AS feature FROM geodata
datahoods(# ) 
datahoods-# INSERT INTO neighborhoods (name, listname, shape_leng, shape_area, the_geometry)
datahoods-# SELECT 
datahoods-#  feature->'properties'->>'name' AS name,
datahoods-#  feature->'properties'->>'listname' AS listname,
datahoods-#  (feature->'properties'->>'shape_leng')::numeric AS shape_leng,
datahoods-#  (feature->'properties'->>'shape_area')::numeric AS shape_area,
datahoods-#  ST_SetSRID(ST_GeomFromGeoJSON(feature->>'geometry'), 4326) as the_geometry
datahoods-#   FROM features;
```

And wow - it works! This is a good start to the visualization that we want to build. Let's perform a couple of queries, just for fun. Here's the five largest features in our imported geojson:
```
datahoods=# SELECT listname, shape_area FROM neighborhoods ORDER BY shape_area DESC LIMIT 5;
  listname  |  shape_area   
------------+---------------
 Somerton   | 129254596.792
 Industrial |  119437430.82
 Bustleton  | 114050423.594
 Airport    | 97738683.7255
 Holmesburg | 84064467.3235
(5 rows)
```

Let's find out what feature contains the cafe I'm writing this at:
```
datahoods=# SELECT listname FROM neighborhoods WHERE ST_Contains(ST_SetSRID(the_geometry, 4326), ST_SetSRID(ST_Point(-75.2021898, 39.9563677) ,4326))=true;
    listname     
-----------------
 University City
(1 row)
```
Note that we have to set the SRIDs of these geometry objects before we can compare them. SRIDs refer to the spatial reference system, and we need to make sure we are comparing apples and apples. Much more on this here: https://en.wikipedia.org/wiki/Spatial_reference_system

That's all for now. In the next post we'll figure out how to add appropriate metadata to our 311 records so that we can use an ORM to query based on their geospatial data.
