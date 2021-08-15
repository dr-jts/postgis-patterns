---
parent: Querying
---

# Query by Distance
{: .no_toc }

1. TOC
{:toc}


## Find points NOT within distance of lines
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

## Find Points which have no point within distance
<https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius>

(Note: question title is misleading, question is actually asking for points which do NOT have a nearby point)
Solution
Best way is to use NOT EXISTS.
To select only records that have no other point with the same value within <threshold> distance:
```sql
SELECT  *
FROM    points AS a
WHERE   NOT EXISTS (
    SELECT  1
    FROM    points
    WHERE   a.cat = cat AND a.id <> id AND ST_DWithin(a.geom, geom, <threshold_in_CRS_units>)
);
```

## Find geometries close to centre of an extent
<https://stackoverflow.com/questions/60218993/postgis-how-do-i-find-results-within-a-given-bounding-box-that-are-close-to-the>
   
## Find Farthest Point from a Polygon
<https://gis.stackexchange.com/questions/332073/is-there-any-function-that-can-calculate-the-maximum-minimum-distance-from-a-geo>

## Find farthest vertex from polygon centroid
<https://stackoverflow.com/questions/31497071/farthest-distance-of-a-polygon-point-from-its-centroid>
   
## Remove Duplicate Points within given Distance
<https://gis.stackexchange.com/questions/24818/remove-duplicate-points-based-on-a-specified-distance>
   
## Find Distance and Bearing from Point to Polygon
https://gis.stackexchange.com/questions/27564/how-to-get-distance-bearing-between-a-point-and-the-nearest-part-of-a-polygon


## Use DWithin instead of Buffer
<https://gis.stackexchange.com/questions/297317/st-intersects-returns-true-while-st-contains-returns-false-for-a-point-located-o>

## Find closest point on boundary of a union of polygons
<https://gis.stackexchange.com/questions/124158/finding-outermost-border-of-set-of-geomertries-circles-using-postgis>

## Find points returned by function within elliptical area
<https://gis.stackexchange.com/questions/17857/finding-points-within-elliptical-area-using-postgis>

## Query Point with highest elevation along a transect through a point cloud
<https://gis.stackexchange.com/questions/223154/find-highest-elevation-along-path>
   
## Query a single point within a given distance of a road
<https://gis.stackexchange.com/questions/361179/postgres-remove-duplicate-rows-returned-by-st-dwithin-query>

## Find Polygons near Lines but not intersecting them
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
