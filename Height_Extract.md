## Processing LiDAR to extract building heights

### Walk through

**Detailed walk through of building extraction using postgis**

First lets pull a data layer from of openstreetmap. You can do this any which way you’d like, as there are a variety of methods for pulling openstreetmap data from their database. Check the [wiki] (http://wiki.openstreetmap.org/wiki/Downloading_data) for a comprehensive list. My favourite method thus far is pulling the data straight into QGIS using the open layers plugin. For those who may want to explore this method, check [this tutorial] (http://www.qgistutorials.com/en/docs/downloading_osm_data.html). For building extraction you only need building footprints, and include the building tags. Not all polygons are of type building in OSM, so we can download all the polygons, and then filter the layer for only polygons tagged as buildings.

LiDAR data was pulled from USGS via the [Earth Explorer site](http://earthexplorer.usgs.gov).
[Here] (http://earthobservatory.nasa.gov/blogs/elegantfigures/2013/05/31/a-quick-guide-to-earth-explorer-for-landsat-8/) is a helpful guide to using the USGS site for those unfamiliar.

Lets start with some of the tools we need, and more importantly, why.

With brew, we are going to install postgres, postgis, and liblas.

[Postgres] (http://www.postgresql.org) is an awesome open source database. It’s really useful for solving the issue of how to manage and process MASSIVE datasets. Simply put, postgres handles tables of data very quickly, and very effectively.

`brew install postgres`

[Postgis] (http://www.postgis.us) is an equally as awesome extension to postgres. This extension allows us to store and manipulate spatial data. Spatial data is stored in a column just like all other types of data, except it is more complex than a number or character. A point is represented as a set of x y coordinates, along with a potential z(elevation) value, and a spatial reference value (SRID), which is an id defining the geometry’s coordinate system, or location on the surface of the earth. Lines and polygons, can get much more complex in their structure. This is why postgis exists. A simple point may be represented as (x,y,z,SRID) in a table cell. If we want to parse through massive columns of complex patterns of spatial data, we need to utilize postgis.

`brew install postgis`

[Liblas] (http://www.liblas.org) is an opensource lidar processing toolkit. A [lidar] (http://en.wikipedia.org/wiki/Lidar) data set has an extension .las. It holds an overwhelming amount of data. How it works: A plane flies over a neighbourhood with a lidar sensor attached to it. This sensor shoots lasers at the surface of the earth. Millions of lasers. These lasers ricochet off objects and by sheer numbers most of them come back to the sensor. The sensor measures the time difference, and calculates an elevation value for the point. These elevation values are relative to eachother, so we need to establish which values are building heights, and which are ground heights. This is what a lidar dataset might look like in qgis:

![denselidarcloud](https://cloud.githubusercontent.com/assets/7583912/3766595/3e2694a6-18c6-11e4-833c-117e06e2f96e.png)

Here is a close up. About 17 million points.

![lidarcloseup](https://cloud.githubusercontent.com/assets/7583912/3766607/55670d94-18c6-11e4-95dd-d0a1ff87b2da.png)


We need liblas to do our initial lidar processing.

`brew install liblas`

Ok, so we have our raw lidar dataset and our building footprints. We will need to preprocess before any analysis.

**Part 1**

Part 1 is going to be pretty dry. Its all prep for greater things to come.

We will:

- create a database and tables
- convert the .las file to a more useable format
- import our lidar data and shape file into our postgres tables.


Cd to your working directory where your .las file is stored. The lasinfo method will return some info on our .las file ….go ahead and take a peek.

`lasinfo SFdowntown.las`

A lot of info is in this file, lets simplify it.

Convert your .las to a text file.

`las2txt -i SFdowntown.las -o SFdowntown.txt`

Open the text file and you will see comma delimited x, y, z values. X and Y define the location of the point on a planar dimension, while z defines the elevation value of the point. Much simpler to understand than the .las format.

Start postgres however you prefer. I use lunchy. You will probably have a different set up.

`lunchy start postgres`

Create a new database.

`createdb SFdowntown`

Lets enter our database and postgres' [psql] (http://www.postgresql.org/docs/9.2/static/app-psql.html). Psql is a SQL like language that allows us to query and understand our database.

`psql SFdowntown`

Activate the postgis extenstion (for spatial objects and functionality).

`CREATE EXTENSION postgis;`

Create a table to hold our lidar dataset.
This creates the infrastructure to hold our x,y,z data. We are naming the table ‘elevation’ because that is primarily what we are interested with in regards to the lidar data.

```sql
CREATE TABLE elevation (x double precision, y double precision, z double precision);
```

Now we will add a geometry column. Postgis documentation suggests to [add a geometry] (http://postgis.refractions.net/docs/AddGeometryColumn.html) column after table creation. If you create the geometry column in the initial table set up, you will have to manually register the geometry in the database. Lets avoid this and follow the documentation. ‘the_geom’ refers to the name of the column, and 32610 is the SRID. This specific SRID is UTM zone 10N. UTM zones are great for dealing with a small area and measurements, which we will be doing. Check out this esri [blog] (http://blogs.esri.com/esri/arcgis/2010/03/05/measuring-distances-and-areas-when-your-map-uses-the-mercator-projection/) for a short summary.

```sql
SELECT AddGeometryColumn ('elevation','the_geom',32610,'POINT',2);
```

Copy the data from our text file to our database. NOTE: if this does not work, re-type the single quotes inside of the terminal.

```sql
copy elevation(x,y,z) FROM ‘~/Desktop/LiDARExtract/SFdowntown/SFdowntown.txt’ DELIMITERS ',';
```

Lets update the elevation table with the ‘geometry from text’ function. This will parse through the table we just copied and pull the coordinates from the x y values. This will locate the points on the planar dimension of our data. This is using the table we just copied in from the previous step. Again using the utm 10n SRID. This will take a couple of minutes.

```sql
UPDATE elevation SET the_geom = ST_GeomFromText('POINT(' || x || ' ' || y || ')',32610);
```

Now that we have our spatial data loaded, we definitely want to create a spatial index. Spatial indices create general envelopes around our geometries, and will allow for quicker query and function abilities later on. Read more about them [here] (http://revenant.ca/www/postgis/workshop/indexing.html).

```sql
CREATE INDEX elev_gix ON elevation USING GIST (the_geom);
```

We will switch our focus over to our shapefile for a second. Let’s exit our database.

```sql
\q
```

Cd to our shapefile directory and create a sql file from our shapefile.

`shp2pgsql -I -s 32610 SFdowntown.shp buildings > buildings.sql`

And load the sql file into our database.

`psql -d SFdowntown -f buildings.sql`

Part 1 done. We have our data preprocessed and bundled up inside of postgres as spatial tables. Lets start manipulating the data with spatial operations to extract some building heights.

**Part 2**

Part 2 gets fun, but challenging. We start performing spatial operations on our tables. We will use QGIS throughout to check our work, to better understand how our data is being changed along the way.

We will:
- Perform spatial joins
- Perform table joins

First off we are going to add a column to our buildings table. This column will store our height values that are recorded from the top of our buildings.

```sql
ALTER TABLE buildings ADD COLUMN height double precision NULL;
```

Lets perform a spatial join. A spatial join is similar to a [table join] (http://www.w3schools.com/sql/sql_join.asp) in a database, except rather than joining two tables based on a common attribute, we are joining information based on a common geography. This is where postgis kicks in. Lets update our buildings table to hold the attributes of overlying lidar points. To control for outliers we will grab the average lidar height values within each individual building.
Note: gid is a unique identifier in our buildings table.

```sql
UPDATE buildings SET height = avgheight FROM (
SELECT y.gid, avg(x.z) As avgheight
FROM elevation x, buildings y
WHERE ST_within(x.the_geom, y.geom) GROUP BY y.gid
) t
WHERE buildings.gid = t.gid;
```

Next we will create a duplicate of our buildings table. We will be using this table to create a 2 metre [buffer] (http://www.gistutor.com/concepts/9-beginner-concept-tutorials/46-gis-buffer.html) around our original buildings table.

```sql
CREATE TABLE buildings2m AS SELECT * FROM buildings;
```

In this step, we will create a 1 metre buffer around our buildings2m table (which is exactly the same as our buildings table), represented by the mustard coloured ring.

![buff1m](https://cloud.githubusercontent.com/assets/7583912/3766619/6dad061a-18c6-11e4-89df-f284230d678f.png)

Great, next we will create a 2 metre buffer around our buildings2m table (which is exactly the same as our buildings table), represented by the red coloured ring.

![buff2m](https://cloud.githubusercontent.com/assets/7583912/3766627/7be009da-18c6-11e4-91c1-7b8180a90862.png)

Lastly, we are going to erase the areas that are shared between the 2m buffer polygons and the 1m buffer polygons. This utilizes the ST_DIFFERENCE command, which accepts the 2m buffer and 1m buffer polygons as arguments below. We are using the the mustard coloured area as a cookie cutter, and are left with a red ring around our buildings.


![cutthecookie](https://cloud.githubusercontent.com/assets/7583912/3766725/71968156-18c7-11e4-8e20-6bb2c9db7033.png)


These steps are bundled up into one psql command, as follows.

```sql
UPDATE buildings2m SET geom = buffer FROM (
SELECT y.gid, st_multi(ST_DIFFERENCE(st_multi(ST_Buffer(y.geom, 2)), st_multi(ST_buffer(y.geom, 1)))) As buffer
FROM buildings2m y
WHERE y.gid > 0
) t
WHERE buildings2m.gid = t.gid;
```

This is our result. We can view this by [bringing the layer](http://www.gistutor.com/quantum-gis/20-intermediate-quantum-gis-tutorials/34-working-with-your-postgis-layers-using-quantum-gis-qgis.html) into qgis.

![rings](https://cloud.githubusercontent.com/assets/7583912/3766634/944fd8b0-18c6-11e4-9433-5ca3042297f2.png)

These rings around our buildings represent the ground level. We will be performing a similar spatial join to that which we executed earlier, except now we will be absorbing the elevation values into the ring shaped polygons. The psql command will go through each ring shaped polygon in our buildings2m spatial table and match up overlying lidar points, grab the minimum value of that overlying collection, and join the value to the height attribute of our ring shaped polygon.

![ringhighlight](https://cloud.githubusercontent.com/assets/7583912/3766691/2bb051f8-18c7-11e4-9776-3412bc57cfa2.png)


You will notice some of the rings overlap with each other and other buildings, we will control for this by extracting the minimum height value, since the lidar lasers cannot record anything lower than the ground level. Furthermore, by extracting the minimum lidar return value, we avoid getting wrong readings that may have ricocheted off of trees, cars, humans, etc. Here is the code.

```sql
UPDATE buildings2m SET height = minheight FROM (
SELECT y.gid, min(x.z) As minheight
FROM elevation x, buildings2m y
WHERE ST_within(x.the_geom, y.geom) GROUP BY y.gid
) t
WHERE buildings2m.gid = t.gid;
```

At this point we have two mutually informative polygons. Our building footprints have a height value representing the return value off of the roof of buildings. Additionally, we have ring shaped polygons around our buildings holding a ground elevation value. Lets join the two in one table and get the difference which will represent our building heights.

Lets add a ground height column to our original buildings table.

```sql
ALTER TABLE buildings ADD COLUMN gheight double precision NULL;
```

Recall that our ring shaped layer (buildings2m) was a copy of our buildings layer. This means it still has its original gid (unique identifier). Lets match these up with the ID’s in our original building tables and carry over our recorded ground heights.

```sql
UPDATE buildings x SET gheight = y.height FROM buildings2m y WHERE x.gid = y.gid;
```

Lastly, lets add a column that will store our final building heights. This will be the difference between our height value and our ground height value.

```sql
ALTER TABLE buildings ADD COLUMN buildheight double precision NULL;
```

Subtract the ground height from the height to get buildheight.

```sql
UPDATE buildings set buildheight = (height - gheight);
```

**Part 3**

We will create a new table named buildings3d to be our final table layer. We will order by the build height, as TileMill requires this to render 3d objects.

```sql
CREATE TABLE buildings3d AS SELECT * FROM buildings ORDER by buildheight;
```

Now, we will transform the coordinate system of this table to the google mercator coordinate system, as until now we have been working in a spatial analysis friendly UTM projection.

```sql
ALTER TABLE buildings3d
ALTER COLUMN geom TYPE geometry(MultiPolygon, 900913) USING ST_Transform (geom, 900913);
```

TileMill can [import layers directly] (https://www.mapbox.com/tilemill/docs/guides/postgis-work/) from postgis.

Lastly, follow [this here tutorial] (https://www.mapbox.com/tilemill/docs/guides/tilemill-complex-3d-structures/) by the friendly folks from Mapbox and you’ll have a 3d render of your map:

![3dmap](https://cloud.githubusercontent.com/assets/7583912/3766645/b852027e-18c6-11e4-87bb-518dded6b220.png)

