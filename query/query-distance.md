---
parent: Querying
---

# Query by Distance
{: .no_toc }

1. TOC
{:toc}

## Nearest queries

### Find Polygons within distance of a Line
<https://gis.stackexchange.com/questions/377674/find-nearest-polygons-of-a-multi-line-string>

Solution also shows ordering result by distance.
```sql
SELECT *
FROM line AS l 
LEFT JOIN polygons AS p 
  ON ST_DWithin(l.geom, p.geom, 200) 
ORDER BY l.geom <-> p.geom;
```

### Find points NOT within distance of lines
<https://gis.stackexchange.com/questions/356497/select-points-falling-outside-of-buffer-and-count>
<https://gis.stackexchange.com/questions/367594/get-all-geom-points-that-are-more-than-3-meters-from-the-linestring-at-big-scal>

#### Solution 1: EXCEPT with DWithin (Fastest)
```sql
SELECT locations.geom FROM locations
EXCEPT 
SELECT locations.geom FROM ways
JOIN locations
ON ST_DWithin(   ways.linestring,    locations.geom,    3)
```
#### Solution 2: LEFT JOIN for non-NULL with DWithin
2x SLOWER than #1
```sql
SELECT  inj.*
   FROM injuries inj 
   LEFT JOIN bike_routes br 
ON ST_DWithin(inj.geom, br.geom, 15) 
   WHERE br.gid IS NULL
```
#### Solution 3: NOT EXISTS with DWithin
Same performance as #2 ?
```sql
SELECT * 
FROM injuries AS inj 
WHERE NOT EXISTS 
(SELECT 1 FROM bike_routes br
WHERE ST_DWithin(br.geom, inj.geom, 15);
```
#### Solution 3: Buffer (Slow)
Buffer line, union, then find all point not in buffer polygon

### Find locations NOT within a distance of multiple features in other table
<https://gis.stackexchange.com/questions/428963/points-which-are-beyond-certain-distance-from-multiple-points>
Find locations beyond a given distance from multiple cities.
  
```sql
SELECT *
FROM   points AS p
WHERE NOT EXISTS (
  SELECT 1
  FROM   cities AS c
  WHERE  ST_DWithin(c.geom, p.geom, <distance>)
    AND  city IN ('New York', 'Washington')
);
```

### Find Points having no other points in table with same value and within distance
<https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius>

**Solution**
Use `NOT EXISTS`.
Select features that do not have a duplicate feature within distance:
  
```sql
SELECT  *
FROM    points AS a
WHERE   NOT EXISTS (
    SELECT  1
    FROM    points
    WHERE   a.val = val AND a.id <> id AND ST_DWithin(a.geom, geom, <distance_in_CRS_units>)
);
```

### Find points within a point-specific Distance
<https://gis.stackexchange.com/questions/473034/postgis-distance-query-using-a-dynamic-radius>

Given a table of points with each record having a `radius` column, 
find all points whose distance from a given fixed point is less than the radius plus a specified distance `QUERY_DIST`.

**Solution:**
Use `ST_Expand` and a functional index.

```sql
-- for GEOMETRY; the radius must be in SRS units
CREATE INDEX ON <table> USING GIST ( ST_Expand(location, radius) );

-- for GEOGRAPHY; radius is in meters
CREATE INDEX ON <table> USING GIST ( _ST_Expand(location, radius) );

SELECT *
FROM <table> AS t
WHERE
  -- note the ST_Expand expression needs to match that of the index definition EXACTLY
  [_]ST_Expand(t.geom, t.radius) && [_]ST_Expand(<QUERY_POINT>, <QUERY_DIST>)
  AND ST_DWithin(
    <QUERY_POINT>,
    t.geom,
    t.radius + <QUERY_DIST>
  );




### Find geometries close to centre of an extent
<https://stackoverflow.com/questions/60218993/postgis-how-do-i-find-results-within-a-given-bounding-box-that-are-close-to-the>
  
   
### Remove Duplicate Points within given Distance
<https://gis.stackexchange.com/questions/24818/remove-duplicate-points-based-on-a-specified-distance>
<https://gis.stackexchange.com/questions/159600/find-all-points-within-5m-with-same-name-on-large-dataset>

No good solutions provided.
   
### Find Distance and Bearing from Point to Polygon
https://gis.stackexchange.com/questions/27564/how-to-get-distance-bearing-between-a-point-and-the-nearest-part-of-a-polygon


### Use DWithin instead of Buffer
<https://gis.stackexchange.com/questions/297317/st-intersects-returns-true-while-st-contains-returns-false-for-a-point-located-o>

### Find nearest point on boundary of a union of polygons
<https://gis.stackexchange.com/questions/124158/finding-outermost-border-of-set-of-geomertries-circles-using-postgis>

### Find points returned by function within elliptical area
<https://gis.stackexchange.com/questions/17857/finding-points-within-elliptical-area-using-postgis>

### Query Point with highest elevation along a transect through a point cloud
<https://gis.stackexchange.com/questions/223154/find-highest-elevation-along-path>
   
### Query a single point within a given distance of a road
<https://gis.stackexchange.com/questions/361179/postgres-remove-duplicate-rows-returned-by-st-dwithin-query>

### Find Polygons near Lines but not intersecting them
<https://gis.stackexchange.com/questions/408256/how-to-select-polygons-that-dont-intersect-with-line-but-with-its-buffer>

Following query includes polygons multiple times if there are multiple lines within distance.
 ```sql
SELECT p.*
FROM polygons p 
  INNER JOIN lines l ON ST_DWithin(p.geom,l.geom, DISTANCE )
WHERE NOT EXISTS (
      SELECT 1
      FROM lines l2 
      WHERE ST_Intersects(p.geom, l2.geom)   
      );
```

To include polygons once only:
```sql
SELECT p.*
FROM polygons p
WHERE EXISTS (
      SELECT 1
      FROM lines l 
      WHERE ST_DWithin(p.geom,l.geom, DISTANCE )  
      )
  AND NOT EXISTS (
      SELECT 1
      FROM lines l2 
      WHERE ST_Intersects(p.geom, l2.geom)   
      );
```

## Farthest queries

### Find furthest pair of locations in groups
<https://stackoverflow.com/questions/70906625/find-the-two-postcodes-furthest-apart-by-district>

Given a set of locations in multiple groups (e.g. postcodes in districts), 
find the pair of locations furthest apart in each group.

Finding the furthest pair of locations requires testing each pair of locations
and selecting the furthest apart. This can be slightly optimized by using a "triangle join",
which evaluates half the total number of pairs by evaluating only pairs where the first item is less than the second item
(assuming the items have an ordered id).

Evaluating this over groups requires using one of the standard SQL patterns to select the first row in a group.
(See <https://stackoverflow.com/questions/3800551/select-first-row-in-each-group-by-group>).

**Solution 1: DISTINCT ON**
```sql
WITH pairs AS (
  SELECT
    loc.district,
    loc.postcode AS postcode1,
    loc2.postcode AS postcode2,
    ST_DistanceSphere( ST_Point( loc.lat, loc.long),
                       ST_Point( loc2.lat,loc2.long) ) AS distance
  FROM locations loc
  LEFT JOIN locations loc2
    ON loc.district = loc2.district
    AND loc.postcode < loc2.postcode  
         -- triangle join compares each pair only once
)
SELECT DISTINCT ON (p.district)
    p.district,
    p.postcode1,
    p.postcode2,
    p.distance
FROM pairs p
ORDER BY p.district, p.distance DESC;
```

**Solution 2: ROW_NUMBER**
```sql
SELECT * 
FROM ( SELECT t1.district, t1.postcode AS postcode1, t2.postcode AS postcode2,
        , row_number() OVER( PARTITION BY t1.district 
                             ORDER BY ST_DistanceSphere(ST_Point(t1.lat, t1.long), ST_Point(t2.lat, t2.long)) desc) rn
       FROM locations t1
       JOIN locations t2 ON t1.district = t2.district AND t1.postcode > t2.postcode
) t
WHERE rn = 1;
```

**Solution 3: LATERAL**

TBD

### Find Farthest Point from a Polygon
<https://gis.stackexchange.com/questions/332073/is-there-any-function-that-can-calculate-the-maximum-minimum-distance-from-a-geo>

```sql
SELECT ST_Distance((st_dumppoints(pts_geom),
    poly.geom) dist
  ) ORDR BY dist desc LIMIT 1
```
  
### Find farthest vertex from polygon centroid
<https://stackoverflow.com/questions/31497071/farthest-distance-of-a-polygon-point-from-its-centroid>
### Find random sample of Point features at least distance D apart
   
* Randomize row order
* Loop over rows
  * build a MultiPoint union of the result
  * add result records if they have distance > D to current result MultiPoint
  * terminate when N records have been found, or when no further points can be added
   
This is fairly reasonable in performance.  For a 2M point table finding 100 different points takes ~ 6 secs.

```sql
WITH RECURSIVE rand AS (
  SELECT geom, name FROM geonames ORDER BY random()
),
pick(count, geomAll, geom, name) AS (
  SELECT 1, geom::geometry AS geomAll, geom::geometry, name 
    FROM (SELECT geom, name FROM rand LIMIT 1) t
  UNION ALL
  SELECT count, ST_Union(geomAll, geom), geom, name
    FROM (SELECT count + 1 AS count, p.geomAll AS geomAll, r.geom, r.name 
              FROM pick p CROSS JOIN rand r
              WHERE ST_Distance(p.geomAll, r.geom) > 1   -- PARAMETER: Distance
              LIMIT 1) t
    WHERE count <= 100.  -- PARAMETER: Result count
)
SELECT count, geom, name FROM pick;
-- Use this to visualize result                     
--SELECT count, ST_AsText(geomAll), ST_AsText(geom), name FROM pick;
```
                      
**Self-contained example:**
                      
```sql
WITH RECURSIVE rand AS (
  SELECT geom, 'row' || path[1] AS name
    FROM ST_Dump( ST_GeneratePoints(ST_MakeEnvelope(0, 0, 100, 100), 10000))
),
pick(count, geomAll, geom, name) AS (
  SELECT 1, geom::geometry AS geomAll, geom::geometry, name
    FROM (SELECT geom, name FROM rand LIMIT 1) t
  UNION ALL
  SELECT count, ST_Union(geomAll, geom), geom, name
    FROM (SELECT count + 1 AS count, p.geomAll AS geomAll, r.geom, r.name 
              FROM pick p CROSS JOIN rand r
              WHERE ST_Distance(p.geomAll, r.geom) > 5
              LIMIT 1) t
    WHERE count <= 100
)
SELECT count, ST_AsText(geomAll), ST_AsText(geom), name FROM pick;
```                     
