---
nav_exclude: true
---

# OLD Overlay
{: .no_toc }

Overlay processes create geometry from existing ones with output vertices being either from the input 
or from intersection points between input line segments.
Often the output geometry is associated with attribution from the inputs which create it.

Overlay and related processing involved using the overlay operations of `ST_Intersection`, `ST_Union`, `ST_Difference` and `ST_SymDifference`,
as well as `ST_Split` and `ST_Node`.

1. TOC
{:toc}


## Noding

### Node (Split) a table of lines by another table of lines
https://lists.osgeo.org/pipermail/postgis-users/2019-September/043617.html

### Node a table of lines with itself
https://gis.stackexchange.com/questions/368996/intersecting-line-at-junction-in-postgresql

### Node a table of Lines by a Set of Points
https://gis.stackexchange.com/questions/332213/split-lines-with-points-postgis

### Compute points of intersection for a LineString
https://gis.stackexchange.com/questions/16347/is-there-a-postgis-function-for-determining-whether-a-linestring-intersects-itse

### Compute location of noding failures
https://gis.stackexchange.com/questions/345341/get-location-of-postgis-geos-topology-exception

Using ST_Node on set of linestrings produces an error with no indication of where the problem occurs.  Currently ST_Node uses IteratedNoder, which nodes up to 6 times, ahd fails if intersections are still found.  
Solution
Would be possible to report the nodes found in the last pass, which wouild indicate where the problems occur.  

Would be better to eliminate noding errors via snap-rounding, or some other kind of snapping

### Clip Set of LineString by intersection points
https://gis.stackexchange.com/questions/154833/cutting-linestrings-with-points

Uses `ST_Split_Multi` from here: https://github.com/Remi-C/PPPP_utilities/blob/master/postgis/rc_split_multi.sql

### Construct locations where LineStrings self-intersect
https://gis.stackexchange.com/questions/367120/getting-st-issimple-reason-and-detail-similar-to-st-isvaliddetail-in-postgis

Solution
SQL to compute LineString self-intersetions is provided



## Line Intersection
### Intersection of Lines which are not exactly coincident
https://stackoverflow.com/questions/60298412/wrong-result-using-st-intersection-with-postgis/60306404#60306404


## Line Merging
### Merge/Node Lines in a linear network
https://gis.stackexchange.com/questions/238329/how-to-break-multilinestring-into-constituent-linestrings-in-postgis
Solution
Use ST_Node, then ST_LineMerge

### Merge Lines and preserve direction
https://gis.stackexchange.com/questions/353565/how-to-join-linestrings-without-reversing-directions-in-postgis

SOLUTION 1
Use a recursive CTE to group contiguous lines so they can be merged

```sql
WITH RECURSIVE
data AS (SELECT
--'MULTILINESTRING((0 0, 1 1), (2 2, 1 1), (2 2, 3 3), (3 3, 4 4))'::geometry
'MULTILINESTRING( (0 0, 1 1), (1 1, 2 2), (3 3, 2 2), (4 4, 3 3), (4 4, 5 5), (5 5, 6 6) )'::geometry
 AS geom)
,lines AS (SELECT t.path[1] AS id, t.geom FROM data, LATERAL ST_Dump(data.geom) AS t)
,paths AS (
  SELECT * FROM
    (SELECT l1.id, l1.geom, l1.id AS startid, l2.id AS previd
      FROM lines AS l1 LEFT JOIN lines AS l2 ON ST_EndPoint(l2.geom) = ST_StartPoint(l1.geom)) AS t
    WHERE previd IS NULL
  UNION ALL
  SELECT l1.id, l1.geom, startid, p.id AS previd
    FROM paths p
    INNER JOIN lines l1 ON ST_EndPoint(p.geom) = ST_StartPoint(l1.geom)
)
SELECT ST_AsText( ST_LineMerge(ST_Collect(geom)) ) AS geom
  FROM paths
  GROUP BY startid;
```

SOLUTION 2
ST_LIneMerge merges lines irrespective of direction.  
Perhaps a flag could be added to respect direction?

See Also
https://gis.stackexchange.com/questions/74119/a-linestring-merger-algorithm

### Merge lines to simplify a road network
Merge lines with common attributes at degree-2 nodes

https://gis.stackexchange.com/questions/326433/st-linemerge-to-simplify-road-network?rq=1


## Line Splitting
### Split Self-Overlapping Lines at Points not on the lines
https://gis.stackexchange.com/questions/347790/splitting-self-overlapping-lines-with-points-using-postgis

### Split Lines by Polygons
https://gis.stackexchange.com/questions/215886/split-lines-by-polygons

## Overlay - Lines
https://gis.stackexchange.com/questions/186242/how-to-get-smallest-line-segments-from-intersection-difference-of-multiple-ove
### Count All Intersections Between 2 Linestrings
https://gis.stackexchange.com/questions/347790/splitting-self-overlapping-lines-with-points-using-postgis

### Remove Line Overlaps Hierarchically
https://gis.stackexchange.com/questions/372572/how-to-remove-line-overlap-hierachically-in-postgis-with-st-difference

## Polygonization
### Polygonize OSM streets
https://gis.stackexchange.com/questions/331529/split-streets-to-create-polygons

### Polygonize a set of lines
https://gis.stackexchange.com/questions/231237/making-linestrings-with-adjacent-lines-in-postgis?rq=1

### Polygonize a set of isolines with bounding box
https://gis.stackexchange.com/questions/127727/how-to-transform-isolines-to-isopolygons-with-postgis

### Find area enclosed by a set of lines
https://gis.stackexchange.com/questions/373933/polygon-covered-by-the-intersection-of-multiple-linestrings-postgis/373983#373983



## Clipping

### Clip Districts to Coastline
http://blog.cleverelephant.ca/2019/07/simple-sql-gis.html

### Clip Unioned Polygons to remove water bodies
https://gis.stackexchange.com/questions/331887/unioning-a-set-of-intersections


## Polygon Intersection

### Aggregated Intersection
<https://gis.stackexchange.com/questions/271824/st-intersection-intersection-of-all-geometries-in-a-table>
<https://gis.stackexchange.com/questions/271941/looping-through-table-to-get-single-intersection-from-n2-geometries-using-postg>
<https://gis.stackexchange.com/questions/60281/query-to-find-the-intersection-coordinates-of-multiple-polyon>
<https://gis.stackexchange.com/questions/269875/aggregate-version-of-st-intersection>

#### Solutions
* Define an aggregate function (given in #2)
* Define a function to do the looping
* Use a recursive CTE (see SQL in #2)

#### Issues
* How to find all groups of intersecting polygons.  DBSCAN maybe?  (This is suggested in an answer)
* Intersection performance - Use Polygons instead of MultiPolygons
  * https://gis.stackexchange.com/questions/101425/using-multipolygon-or-polygon-features-for-large-intersect-operations

### Intersection performance - Check containment first
https://postgis.net/2014/03/14/tip_intersection_faster/



## Polygon Difference

### Subtract large set of polygons from a surrounding box
https://gis.stackexchange.com/questions/330051/obtaining-the-geospatial-complement-of-a-set-of-polygons-to-a-bounding-box-in-po/333562#333562

#### Issues
conventional approach is too slow to use  (Note: user never actually completed processing, so might not have encountered geometry size issues, which could also occur)

### Subtract MultiPolygons from LineStrings
https://gis.stackexchange.com/questions/239696/subtract-multipolygon-table-from-linestring-table

https://gis.stackexchange.com/questions/11592/difference-between-two-layers-in-postgis
#### Solution
```sql
SELECT COALESCE(ST_Difference(river.geom, lakes.geom), river.geom) As river_geom 
FROM river 
  LEFT JOIN lakes ON ST_Intersects(river.geom, lakes.geom);
```

https://gis.stackexchange.com/questions/193217/st-difference-on-linestrings-and-polygons-slow-and-fails



#### Solution
```sql
SELECT row_number() over() AS gid,
ST_CollectionExtract(ST_Multi(ST_Difference(a.geom, b.geom)), 2)::geometry(MultiLineString, 27700) as geom
FROM lines a
  JOIN LATERAL (
    SELECT ST_UNION(polygons.geom)
    FROM polygons
    WHERE ST_Intersects(a.geom,polygons.geom)
  ) AS b;
```

### Split Polygons by distance from a Polygon
https://gis.stackexchange.com/questions/78073/separate-a-polygon-in-different-polygons-depending-of-the-distance-to-another-po

### Cut Polygons into a Polygonal coverage
https://gis.stackexchange.com/questions/71461/using-st-difference-and-preserving-attributes-in-postgis

#### Solution
For each base polygon, union all detailed polygons which intersect it
Difference the detailed union from the each base polygon
UNION ALL:
The differenced base polygons
The detailed polygons
All remaining base polygons which were not changed

### Subtract Areas from a set of Polygons
https://gis.stackexchange.com/questions/250674/postgis-st-difference-similar-to-arcgis-erase

https://gis.stackexchange.com/questions/187406/how-to-use-st-difference-and-st-intersection-in-case-of-multipolygons-postgis

https://gis.stackexchange.com/questions/90174/postgis-when-i-add-a-polygon-delete-overlapping-areas-in-other-layers

https://gis.stackexchange.com/questions/155597/using-st-difference-to-remove-overlapping-features

```sql
SELECT id, COALESCE(ST_Difference(geom, (SELECT ST_Union(b.geom) 
                                         FROM parcels b
                                         WHERE ST_Intersects(a.geom, b.geom)
                                         AND a.id != b.id)), 
                    a.geom)
FROM parcels a;
```

### Find Part of Polygons not fully contained by union of other Polygons
Find what countries are not fully covered by administrative boundaries and the geometry of part of country geometry 
where it is not covered by the administrative boundaries.

https://gis.stackexchange.com/questions/313039/find-what-polygons-are-not-fully-covered-by-union-of-polygons-from-another-layer

![](https://i.stack.imgur.com/0kFJj.png)

### Remove overlaps by lower-priority polygons
https://gis.stackexchange.com/questions/379300/how-to-remove-overlaps-and-keep-highest-priority-polygon

#### Solution

```sql
SELECT ST_Multi(COALESCE(
         ST_Difference(a.geom, blade.geom),
         a.geom
       )) AS geom
FROM   table1 AS a
CROSS JOIN LATERAL (
  SELECT ST_Union(geom) AS geom
  FROM   table1 AS b
  WHERE  a.prio > b.prio
) AS blade;
```

![](https://i.stack.imgur.com/W326R.png)



## Polygon Symmetric Difference

### Construct symmetric difference of two tables
https://gis.stackexchange.com/questions/302458/symmetrical-difference-between-two-layers




## Polygon Union

### Union Polygons from two tables, grouped by name
https://gis.stackexchange.com/questions/378699/merging-two-multipolygon-tables-into-one-and-afterwards-dissolve-boundaries
```sql
SELECT name, ST_Multi(ST_Union(geom)) AS geom
FROM ( 
  SELECT name, geom FROM table1
  UNION ALL
  SELECT name, geom FROM table2 ) q
GROUP BY name;
```

### Union of Massive Number of Point Buffers
https://gis.stackexchange.com/questions/31880/memory-issue-when-trying-to-buffer-union-large-dataset-using-postgis?noredirect=1&lq=1

Union a massive number of buffers around points which have an uneven distribution (points are demographic data in the UK).
Using straight ST_Union runs out of memory
Solution
Implement a “SQL-level” cascaded union as follows:
Spatially sort data based on ST_GeoHash
union smaller partitions of the data (partition size = 100K)
union the partitions together 

### Polygon Coverage Union with slivers removed
https://gis.stackexchange.com/questions/71809/is-there-a-dissolve-st-union-function-that-will-close-gaps-between-features?rq=1

Solution - NOT SURE

### Polygon Coverage Union with gaps removed
https://gis.stackexchange.com/questions/356480/is-it-possible-create-a-polygon-from-more-polygons-that-are-not-overlapped-but-c

https://gis.stackexchange.com/questions/316000/using-st-union-to-combine-several-polygons-to-one-multipolygon-using-postgis/31612

Both of these have answers recommending using a small buffer outwards and then the inverse on the result.

### Union Intersecting Polygons
https://gis.stackexchange.com/questions/187728/alternative-to-st-union-st-memunion-for-merging-overlapping-polygons-using-postg?rq=1

### Union groups of polygons
https://gis.stackexchange.com/questions/185393/what-is-the-best-way-to-merge-lots-of-small-adjacents-polygons-postgis?noredirect=1&lq=1

### Union Edge-Adjacent Polygons
Only union polygons which share an edge (not just touch)
https://gis.stackexchange.com/questions/1387/is-there-a-dissolve-function-in-postgis-other-than-st-union?rq=1
https://gis.stackexchange.com/questions/24634/merging-polygons-that-intersect-by-more-than-a-specified-amount?rq=1
https://gis.stackexchange.com/questions/127019/merge-any-and-all-adjacent-polygons?noredirect=1&lq=1
Problem
Union only polygons which intersect, keep non-intersecting ones unchanged.  Goal is to keep attributes on non-intersecting polygons, and improve performance by unioning only groups of intersecting polygons
Solution
Should be able to find equivalence classes of intersecting polygons and union each separately?
See Also

### Grouping touching Polygons
Can use ST_DBSCAN with very small distance to group touching polygons

### Enlarge Polygons to Fill Boundary
https://gis.stackexchange.com/questions/91889/adjusting-polygons-to-boundary-and-filling-holes?rq=1

### Boundary of Coverage of Polygons
https://gis.stackexchange.com/questions/324736/extracting-single-boundary-from-multipolygon-in-postgis

Solution
The obvious: Union, then extract boundary

### Union of cells grouped by ID
https://gis.stackexchange.com/questions/288880/finding-geometry-of-cluster-from-points-collection-using-postgis

### Union of set of geometry specified by IDs
SELECT ST_Union(geo)) FROM ( SELECT geom FROM table WHERE id IN ( … ) ) as foo;

### Union of polygons with equal or lower value
https://gis.stackexchange.com/questions/161849/postgis-sql-request-with-aggregating-st-union?rq=1
Solution
Nice use of window functions with PARTITION BY and ORDER BY.
Not sure what happens if there are two polygons with same value though.  Worth finding out

### Union Groups of Adjacent Polygon, keeping attribution for singletons
https://gis.stackexchange.com/questions/366374/how-to-use-dissolve-a-subset-of-a-postgis-table-based-on-a-value-in-a-column

Solution
Use ST_ClusterDBSCAN

### Union Non-clean Polygons
https://gis.stackexchange.com/questions/31895/joining-lots-of-small-polygons-to-form-larger-polygon-using-postgis

![](https://i.stack.imgur.com/5P53M.png)

#### Solution - Snap Polygons to grid to help clean invalid polygons 
```sql
 SELECT ST_Union(ST_SnapToGrid(the_geom,0.0001)) 
 FROM parishes
 GROUP BY county_name;
```
### Create polygons that fill gaps
https://gis.stackexchange.com/questions/60655/creating-polygons-from-gaps-in-postgis

![](https://i.stack.imgur.com/PjcTl.png)

```sql
WITH polygons(geom) AS
(VALUES (ST_Buffer(ST_Point(0, 0), 1.1,3)),
        (ST_Buffer(ST_Point(0, 2), 1.1,3)),
        (ST_Buffer(ST_Point(2, 2), 1.1,3)),
        (ST_Buffer(ST_Point(2, 0), 1.1,3)),
        (ST_Buffer(ST_Point(4, 1), 1.3,3))
),
bigpoly AS
(SELECT ST_UNION(geom)geom 
 FROM polygons)
SELECT ST_BuildArea(ST_InteriorRingN(geom,i)) 
FROM bigpoly
CROSS JOIN generate_series(1,(SELECT ST_NumInteriorRings(geom) FROM bigpoly)) as i;
```

## Polygon Splitting

https://gis.stackexchange.com/questions/299849/split-polygon-into-separate-polygons-using-a-table-of-individual-lines

### Split rectangles along North-South axis
https://gis.stackexchange.com/questions/239801/how-can-i-split-a-polygon-into-two-equal-parts-along-a-n-s-axis?rq=1

### Split rotated rectangles into equal parts
https://gis.stackexchange.com/questions/286184/splitting-polygon-in-equal-parts-based-on-polygon-area

See also next problem

### Split Polygons into equal-area parts
http://blog.cleverelephant.ca/2018/06/polygon-splitting.html

Does this really result in equal-area subdivision? The Voronoi-of-centroid step is distance-based, not area based…. So may not always work?  Would be good to try this on a bunch of country outines

### Splitting by Line creates narrow gap in Polygonal coverage
https://gis.stackexchange.com/questions/378705/postgis-splitting-polygon-with-line-returns-different-size-polygons-that-creat

Cause is that vertex introduced by splitting is not present in adjacent polygon.

> Perhaps differencing splitting line from surrounding intersecting polygon would introduce that vertex?  
> Or is it better to snap surrounding polygons to split vertices? Probably need a complex process to do this - not really something that can be done easily in DB?

### Splitting Polygon creates invalid coverage with holes
https://gis.stackexchange.com/questions/344716/holes-after-st-split-with-postgis

#### Solution
None so far. Need some way to create a clean coverage


## Overlay Polygons

https://gis.stackexchange.com/questions/109692/how-to-replicate-arcgis-intersect-in-postgis

http://blog.cleverelephant.ca/2019/07/postgis-overlays.html

### Flatten / Create coverage from Nested Polygons
https://gis.stackexchange.com/questions/266005/postgis-separate-nested-polygons

### Create Coverage from overlapping Polygons
https://gis.stackexchange.com/questions/83/separate-polygons-based-on-intersection-using-postgis
https://gis.stackexchange.com/questions/112498/postgis-overlay-style-union-not-dissolve-style
Solution
One answer suggests the standard Extract Lines > Node > Polygonize approach (although does not include the PIP parentage step).  But a comment says that this does not scale well (Pierre Racine…).
Also links to PostGIS wiki:  https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables

### Improve performance of a coverage overlay
https://gis.stackexchange.com/questions/31310/acquiring-arcgis-like-speed-in-postgis/31562
Problem
Finding all intersections of a large set of parcel polygons against a set of jurisdiction polygons is slow
Solution
Reduce # calls to ST_Intersection by testing if parcel is wholly contained in polygon. 
```sql
INSERT INTO parcel_jurisdictions(parcel_gid,jurisdiction_gid,isect_geom) 
  SELECT a.orig_gid AS parcel_gid, b.orig_gid AS jurisdiction_gid, 
    CASE WHEN ST_Within(a.geom,b.geom) THEN a.geom ELSE ST_Multi(ST_Intersection(a.geom,b.geom)) END AS geom 
  FROM valid_parcels a 
    JOIN valid_jurisdictions b ON ST_Intersects(a.geom, b.geom);
```
References
https://postgis.net/2014/03/14/tip_intersection_faster/


### Find non-covered polygons
https://gis.stackexchange.com/questions/333302/selecting-non-overlapping-polygons-from-a-one-layer-in-postgis/334217
```sql
WITH
data AS (
    SELECT * FROM (VALUES
        ( 'A', 'POLYGON ((100 200, 200 200, 200 100, 100 100, 100 200))'::geometry ),
        ( 'B', 'POLYGON ((300 200, 400 200, 400 100, 300 100, 300 200))'::geometry ),
        ( 'C', 'POLYGON ((100 400, 200 400, 200 300, 100 300, 100 400))'::geometry ),
        ( 'AA', 'POLYGON ((120 380, 180 380, 180 320, 120 320, 120 380))'::geometry ),
        ( 'BA', 'POLYGON ((110 180, 160 180, 160 130, 110 130, 110 180))'::geometry ),
        ( 'BB', 'POLYGON ((170 130, 190 130, 190 110, 170 110, 170 130))'::geometry ),
        ( 'CA', 'POLYGON ((330 170, 380 170, 380 120, 330 120, 330 170))'::geometry ),
        ( 'AAA', 'POLYGON ((330 170, 380 170, 380 120, 330 120, 330 170))'::geometry ),
        ( 'BAA', 'POLYGON ((121 171, 151 171, 151 141, 121 141, 121 171))'::geometry ),
        ( 'CAA', 'POLYGON ((341 161, 351 161, 351 141, 341 141, 341 161))'::geometry ),
        ( 'CAB', 'POLYGON ((361 151, 371 151, 371 131, 361 131, 361 151))'::geometry )
    ) AS t(id, geom)
)
SELECT a.id
FROM data AS A
LEFT JOIN data AS b ON a.id <> b.id AND ST_CoveredBy(a.geom, b.geom)
WHERE b.geom IS NULL;
```
### Count Overlap Depth in set of polygons
https://gis.stackexchange.com/questions/159282/counting-overlapping-polygons-in-postgis-using-st-union-very-slow

Solution 1: 
Compute overlay of dataset using ST_Node and ST_Polygonize
Count overlap depth using ST_PointOnSurface and ST_Contains

### Identify Overlay Resultant Parentage

https://gis.stackexchange.com/questions/315368/listing-all-overlapping-polygons-using-postgis

### Union of two polygon layers
Wants a coverage overlay (called “QGIS Union”)
https://gis.stackexchange.com/questions/302086/postgis-union-of-two-polygons-layers

See also
https://gis.stackexchange.com/questions/179533/arcgis-union-equivalent-in-postgis
https://gis.stackexchange.com/questions/115927/is-there-a-union-function-for-multiple-layers-comparable-to-arcgis-in-open-sourc


### Sum of polygonal grid values weighted by area of intersection with polygon
https://gis.stackexchange.com/questions/171333/weighting-amount-of-overlapping-polygons-in-postgis

### Sum Percentage of Polygons intersected by other Polygons, by label

Given two polygonal coverages A and B with labels, compute the percentage area of each polygon in A covered by each label in B 
https://gis.stackexchange.com/questions/378532/working-out-percentage-of-polygon-covering-another-using-postgis


![](https://i.stack.imgur.com/bQjSi.png)

```sql
WITH poly_a(name, geom) AS (VALUES
    ( 'a1', 'POLYGON ((100 200, 200 200, 200 100, 100 100, 100 200))'::geometry ),
    ( 'a2', 'POLYGON ((300 300, 300 200, 200 200, 200 300, 300 300))'::geometry ),
    ( 'a3', 'POLYGON ((400 400, 400 300, 300 300, 300 400, 400 400))'::geometry )
),
poly_b(name, geom) AS (VALUES
    ( 'b1', 'POLYGON ((120 280, 280 280, 280 120, 120 120, 120 280))'::geometry ),
    ( 'b2', 'POLYGON ((280 280, 280 320, 320 320, 320 280, 280 280))'::geometry ),
    ( 'b2', 'POLYGON ((390 390, 390 360, 360 360, 360 390, 390 390))'::geometry )
)
SELECT a.name, b.name, SUM( ST_Area(ST_Intersection(a.geom, b.geom))/ST_Area(a.geom) ) pct
FROM poly_a a JOIN poly_b b ON ST_Intersects(a.geom, b.geom)
GROUP BY a.name, b.name;
```


### Remove Polygons overlapped area on update to Polygon in other table 
https://gis.stackexchange.com/questions/90174/postgis-when-i-add-a-polygon-delete-overlapping-areas-in-other-layers

### Return only polygons from Overlay
https://gis.stackexchange.com/questions/89231/postgis-st-intersection-of-polygons-can-return-lines

https://gis.stackexchange.com/questions/242741/st-intersection-returns-erroneous-polygons

### Compute Coverage from Overlapping Polygons
https://gis.stackexchange.com/questions/206473/obtaining-each-unique-area-of-overlapping-polygons-in-postgres-9-6-postgis-2-3

Problem
Reduce a dataset of highly overlapping polygons to a coverage (not clear if attribution is needed or not)

Issues
User implements a very complex overlay process, but can not get it to work, likely due to robustness problems

Solution
ST_Boundary -> ST_Union -> ST_Polygonize ??


