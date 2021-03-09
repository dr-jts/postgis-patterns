## PostGIS Patterns - Query
{: .no_toc }

1. TOC
{:toc}

## Query Point in Polygon
### Find Points contained by Polygons with attributes
https://gis.stackexchange.com/questions/354319/how-to-extract-attributes-of-polygons-at-specific-points-into-new-point-layer-in

#### Solution
A simple query to do this is:
```sql
SELECT pt.id, poly.*
  FROM grid pt
  JOIN polygons poly ON ST_Intersects(poly.geom, pt.geom);
```
Caveat: this will return multiple records if a point lies in multiple polygons. To ensure only a single record is returned per point, and also to include points which do not lie in any polygon, use:
```sql
SELECT pt.id, poly.*
  FROM grid pt
  LEFT OUTER JOIN LATERAL 
    (SELECT * FROM polygons poly 
       WHERE ST_Intersects(poly.geom, pt.geom) LIMIT 1) AS poly ON true;
```

### Count kinds of Points in Polygons
https://gis.stackexchange.com/questions/356976/postgis-count-number-of-points-by-name-within-distinct-polygons
```sql
SELECT
  polyname,
  count(pid) FILTER (WHERE pid='w') AS "w",
  count(pid) FILTER (WHERE pid='x') AS "x",
  count(pid) FILTER (WHERE pid='y') AS "y",
  count(pid) FILTER (WHERE pid='z') AS "z"
FROM polygons
    LEFT JOIN points ON st_intersects(points.geom, polygons.geom)
GROUP BY polyname;
```
### Optimize Point-in-Polygon query by evaluating against smaller polygons
Count lightning occurences inside countries

https://gis.stackexchange.com/questions/83615/optimizing-st-within-query-to-count-lightning-occurrences-inside-country

### Optimize Point-in-Polygon query by gridding polygons
Count occurences inside river polygons

https://gis.stackexchange.com/questions/185381/optimising-a-very-large-point-in-polygon-query

### Find smallest Polygon containing Point
https://gis.stackexchange.com/questions/220313/point-within-a-polygon-within-another-polygon

![](https://i.stack.imgur.com/vFDCs.png)

#### Solution
Choose containing polygon with smallest area 
```sql
SELECT DISTINCT ON (compequip.id), compequip.*, a.*
FROM compequip
LEFT JOIN a
ON ST_within(compequip.geom, a.geom)
ORDER BY compequip.id, ST_Area(a.geom)
```
### Find Points NOT in Polygons
https://gis.stackexchange.com/questions/139880/postgis-st-within-or-st-disjoint-performance-issues

https://gis.stackexchange.com/questions/26156/updating-attribute-values-of-points-outside-area-using-postgis

https://gis.stackexchange.com/questions/313517/postgresql-postgis-spatial-join-but-keep-all-features-that-dont-intersect

This is not PiP, but the solution of using NOT EXISTS might be applicable?

https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table

### Find Point in Polygon with greatest attribute
Given 2 tables:

* `obstacles` - Point layer with a column `height_m INTEGER` 
* `polyobstacles` - Polygon layer

Select the highest obstacle in each polygon. If there are several points with the same highest height a random one of those is selected.

#### Solution - JOIN LATERAL
```sql
SELECT poly.id, obs_max.*
FROM polyobstacle poly 
JOIN LATERAL (SELECT * FROM obstacles o
  WHERE ST_Contains(poly.geom, o.geom) 
 ORDER BY height_m LIMIT 1
  ) AS obs_max ON true;
```
#### Solution - DISTINCT ON
Do a spatial join between polygon and points and use `SELECT DISTINCT ON (poly.id) poly.id, o.height...`

#### Solution - ARRAY_AGG
```sql
SELECT p.id, (array_agg(o.id order by height_m))[1] AS highest_id
FROM polyobstacles p JOIN obstacles o ON ST_Contains(p.geom, o.geom)
GROUP BY p.id;
```
### Find Polygon containing Point
Basic query - with tables of address points and US census blocks, find state for each point
Discusses required indexes, and external parallelization
https://lists.osgeo.org/pipermail/postgis-users/2020-May/044161.html

### Count Points in Polygons with two Point tables
https://gis.stackexchange.com/questions/377741/find-count-of-multiple-tables-that-st-intersect-a-main-table

```sql
SELECT  ply.polyname, SUM(pnt1.cnt) AS pointtable1count, SUM(pnt2.cnt) AS pointtable2count
FROM    polytable AS ply,
        LATERAL (
          SELECT COUNT(pt.*) AS cnt
          FROM   pointtable1 AS pt
          WHERE  ST_Intersects(ply.geom, pt.geom)
        ) AS pnt1,
        LATERAL (
          SELECT COUNT(c.*) AS cnt
          FROM   pointtable2 AS pt
          WHERE  ST_Intersects(ply.geom, pt.geom)
        ) AS pnt2
GROUP BY 1;
```


## Query Lines

### Find Lines which have a given angle of incidence
https://gis.stackexchange.com/questions/134244/query-road-shape?noredirect=1&lq=1

### Find Line Intersections
https://gis.stackexchange.com/questions/20835/identifying-road-intersections-using-postgis

### Find Lines which intersect N Polygons
https://gis.stackexchange.com/questions/349994/st-intersects-with-multiple-geometries

#### Solution
Nice application of counting using `HAVING` clause.

```sql
SELECT lines.id, lines.geom 
FROM lines
 JOIN polygons ON st_intersects(lines.geom,polygons.geom)
WHERE polygons.id in (1,2)
GROUP BY lines.id, lines.geom 
HAVING count(*) = 2;
```
### Count number of intersections between Line Segments
https://gis.stackexchange.com/questions/365575/counting-geometry-intersections-between-two-linestrings

### Find Begin and End of circular sublines
https://gis.stackexchange.com/questions/206815/seeking-algorithm-to-detect-circling-and-beginning-and-end-of-circle

### Find Longest Line Segment
https://gis.stackexchange.com/questions/359825/get-the-maximum-distance-between-two-consecutive-points-in-a-linestring

### Find non-monotonic Z ordinates in a LineString
https://gis.stackexchange.com/questions/374459/postgis-validate-the-z-continuity

Assuming the LineStrings are digitized in the correct order (start point is most elevated),
this returns all vertices geom, their respective line <id>, and their position in the vertices array of that line, 
for all vertices with higher Z value as their predecessor.
  
```sql
SELECT ln_id, vtx_id, geom
FROM (SELECT ln.<id> AS ln_id,
         dmp.path[1] AS vtx_id,
         dmp.geom,
         ST_Z(dmp.geom) < LAG(ST_Z(dmp.geom)) OVER(PARTITION BY ln.<id> ORDER BY dmp.path[1]) AS is_valid
  FROM   <lines> AS ln,
         LATERAL ST_DumpPoints(ln.geom) AS dmp
) q
WHERE NOT is_valid;
```

  
## Query Polygons

### Find polygons intersecting other table of polygons using ST_Subdivide
https://gis.stackexchange.com/questions/224138/postgis-st-intersects-vs-arcgis-select-by-location

## Query Intersects

### Find geometries in a table which do NOT intersect another table
https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table

Use NOT EXISTS:
```sql
SELECT * FROM polygons
WHERE NOT EXISTS (SELECT 1 FROM streets WHERE ST_Intersects(polygons.geom, streets.geom))
```
## Query Spatial Relationship

### Find Polygons not contained by other Polygons
https://gis.stackexchange.com/questions/185308/find-polygons-that-does-not-contain-any-polygons-with-postgis

#### Solution
Use the LEFT JOIN on `ST_Contains` with NULL result pattern

### Find Polygons NOT covered by union of other Polygons
https://gis.stackexchange.com/questions/313039/find-what-polygons-are-not-fully-covered-by-union-of-polygons-from-another-layer

### Find Polygons covered by a set of other polygons
https://gis.stackexchange.com/questions/212543/compare-row-to-all-others-in-postgis

#### Solution
For each polygon, compute union of polygons which intersect it, then test if the union covers the polygon

### Find Polygons which touch in a line
```sql
WITH 
data(id, geom) AS (VALUES
    ( 1, 'POLYGON ((100 200, 200 200, 200 100, 100 100, 100 200))'::geometry ),
    ( 2, 'POLYGON ((250 150, 200 150, 200 250, 250 250, 250 150))'::geometry ),
    ( 3, 'POLYGON ((300 100, 250 100, 250 150, 300 150, 300 100))'::geometry )
)
SELECT a.id, b.id, 
    ST_Relate(a.geom, b.geom),
    ST_Relate(a.geom, b.geom, '****1****') AS is_line_touch
    FROM data a CROSS JOIN data b WHERE a.id < b.id;
```

### Find Polygons in a Coverage NOT fully enclosed by other Polygons
https://gis.stackexchange.com/questions/291824/determine-if-a-polygon-is-not-enclosed-by-other-polygons

![](https://i.stack.imgur.com/tp5WK.png)

#### Solution
Find polygons where total length of intersection with others is less than length of boundary

```sql
SELECT a.id
FROM data a 
INNER JOIN data b ON (ST_Intersects(a.geom, b.geom) AND a.id != b.id) 
GROUP BY a.id
HAVING 1e-6 > 
  abs(ST_Length(ST_ExteriorRing(a.geom)) - 
  sum(ST_Length(ST_Intersection(ST_Exteriorring(a.geom), ST_ExteriorRing(b.geom)))));
```

### Test if Point is on a Line
https://gis.stackexchange.com/questions/11510/st-closestpointline-point-does-not-intersect-line

Also https://gis.stackexchange.com/questions/350461/find-path-containing-point

### Find Start Points of Rivers and Headwater polygons
https://gis.stackexchange.com/questions/131806/find-start-of-river
https://gis.stackexchange.com/questions/132266/find-headwater-polygons?noredirect=1&lq=1

### Find routes which terminate in Polygons but do not cross them
https://gis.stackexchange.com/questions/254051/selecting-lines-with-start-and-end-points-inside-polygons-but-do-not-cross-them

### Find Lines that touch Polygon at both ends
https://gis.stackexchange.com/questions/299319/select-only-lines-that-touch-both-sides-of-polygon-postgis

### Find Lines that touch but do not cross Polygons
https://gis.stackexchange.com/questions/160142/intersection-between-line-polygon-in-postgis
```sql
SELECT lines.geom
 FROM lines, polygons
 WHERE ST_Touches(lines.geom, polygons.geom) AND
                 NOT EXISTS (SELECT 1 FROM polygons p2 WHERE ST_Crosses(lines.geom, p2.geom));
```
### Compute hierarchy of a nested Polygonal Coverage
https://gis.stackexchange.com/questions/343100/intersecting-polygons-to-build-boundary-hierearchy-using-postgis

A table of polygons which form a set of nested hierarchical coverages, but coverage hierarchy is not explicitly represented.
#### Solution
Determine contains relationships based on interior points and areas. Can use a recursive query on that to extract paths if needed. 

```sql
WITH RECURSIVE data(id, geom) AS (VALUES
('AC11', 'POLYGON ((100 200, 150 200, 150 150, 100 150, 100 200))'),
('AC12', 'POLYGON ((200 200, 200 150, 150 150, 150 200, 200 200))'),
('AC21', 'POLYGON ((200 100, 150 100, 150 150, 200 150, 200 100))'),
('AC22', 'POLYGON ((100 100, 100 150, 150 150, 150 100, 100 100))'),
('AC1', 'POLYGON ((200 200, 200 150, 100 150, 100 200, 200 200))'),
('AC2', 'POLYGON ((200 100, 100 100, 100 150, 200 150, 200 100))'),
('AC', 'POLYGON ((100 200, 200 200, 200 100, 100 100, 100 200))'),
('AB1', 'POLYGON ((100 300, 150 300, 150 200, 100 200, 100 300))'),
('AB2', 'POLYGON ((200 300, 200 200, 150 200, 150 300, 200 300))'),
('AB', 'POLYGON ((100 300, 200 300, 200 200, 100 200, 100 300))'),
('AA', 'POLYGON ((0 300, 100 300, 100 100, 0 100, 0 300))'),
('A', 'POLYGON ((200 100, 0 100, 0 300, 200 300, 200 100))')
),
-- compute all containment links
contains AS ( SELECT p.id idpar, c.id idch, ST_Area(p.geom) par_area
    FROM data p 
    JOIN data c ON ST_Contains(p.geom, ST_PointOnSurface(c.geom))
    WHERE ST_Area(p.geom) > ST_Area(c.geom)
),
-- extract direct containment links, by choosing parent with min area
pcrel AS ( SELECT DISTINCT ON (idch) idpar, idch 
    FROM contains ORDER BY idch, par_area ASC
),
-- compute paths as strings
pcpath(id, path) AS (
    SELECT 'A' AS id, 'A' AS path
    UNION ALL
    SELECT idch AS id, path || ',' || idch
        FROM pcpath JOIN pcrel ON pcpath.id = pcrel.idpar
)
SELECT * FROM pcpath;
```

### Test if two 3D geometries are equal
https://gis.stackexchange.com/questions/373978/how-to-check-two-3d-geometry-are-equal-in-postgis/373980#373980

### Find LineStrings with Common Segments
https://gis.stackexchange.com/questions/268147/find-linestrings-with-common-segments-in-postgis-2-3
```sql
SELECT ST_Relate('LINESTRING(0 0, 2 0)'::geometry,
                 'LINESTRING(1 0, 2 0)'::geometry,
                 '1********');
```                 


## Query Spatial Statistics

### Count Points contained in Polygons

#### Solution - LATERAL
Good use case for JOIN LATERAL
```sql
SELECT bg.geoid, bg.geom,  bg.total_pop AS total_population, 
       bg.med_inc AS median_income,
       t.numbirds
  FROM bg_pop_income bg
  JOIN LATERAL
        (SELECT COUNT(1) as numbirds 
           FROM  bird_loc bl 
           WHERE ST_within(bl.loc, bg.geom)) AS t ON true;
```
#### Solution using GROUP BY
Almost certainly less performant
```sql
SELECT bg.geoid, bg.geom, bg.total_pop, bg.med_inc, 
       COUNT(bl.global_unique_identifier) AS num_bird_counts
  FROM  bg_pop_income bg 
  LEFT OUTER JOIN bird_loc bl ON ST_Contains(bg.geom, bl.loc)
  GROUP BY bg.geoid, bg.geom, bg.total_pop, bg.med_inc;
```
### Count points from two tables which lie inside polygons
https://stackoverflow.com/questions/59989829/count-number-of-points-from-two-datasets-contained-by-neighbourhood-polygons-u

### Count polygons which lie within two layers of polygons
https://gis.stackexchange.com/questions/115881/how-many-a-features-that-fall-both-in-the-b-polygons-and-c-polygons-are-for-each

***(Question is for ArcGIS; would be interesting to provide a PostGIS answer)***

### Count number of adjacent polygons in a coverage
<https://gis.stackexchange.com/questions/389448/postgis-count-adjacent-polygons-connected-by-line>
(First query)
```sql
SELECT a.id, a.geom, a.codistat, a.name, num_adj
FROM municipal a 
JOIN LATERAL (SELECT COUNT(1) num_adj 
              FROM municipal b
              WHERE ST_Intersects(a.geom, b.geom)
              ) t ON true;
```

### Count number of adjacent polygons in a coverage connected by a line
<https://gis.stackexchange.com/questions/389448/postgis-count-adjacent-polygons-connected-by-line>
```sql
SELECT a.id, a.geom, a.codistat, a.name, num_adj
FROM municipal a 
JOIN LATERAL (SELECT COUNT(1) num_adj 
              FROM municipal b
              JOIN way w
              ON ST_Intersects(b.geom, w.geom)
              WHERE ST_Intersects(a.geom, b.geom)
                 AND ST_Intersects(a.geom, w.geom)
              ) t ON true;
```

### Find Median of values in a Polygon neighbourhood in a Polygonal Coverage
<https://gis.stackexchange.com/questions/349251/finding-median-of-polygons-that-share-boundaries>

### Sum length of Lines intersecting Polygons
```sql
SELECT county, SUM(ST_Length(ST_Intersection(counties.geom,routes.geom)))
FROM counties
JOIN routes ON ST_Intersects(counties.geom, routes.geom)
GROUP BY county;
```
See following (but answers are not great)
<https://gis.stackexchange.com/questions/143438/calculating-total-line-lengths-within-polygon>


## Query using Index
https://gis.stackexchange.com/questions/172266/improve-performance-of-a-postgis-st-dwithin-query

https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table

### Use ST_Intersects instead of ST_DIsjoint
https://gis.stackexchange.com/questions/167945/delete-polygonal-features-in-one-layer-outside-polygon-feature-of-another-layer

### Spatial Predicate argument ordering important for indexing
https://gis.stackexchange.com/questions/209959/how-to-use-st-intersects-with-different-geometry-type

This may be due to indexing being used for first ST_Intersects arg, but not second?

### Clustering on a spatial Index
https://gis.stackexchange.com/questions/240721/postgis-performance-increase-with-cluster

### Use an Index for performance
https://gis.stackexchange.com/questions/237709/speeding-up-intersect-query-in-postgis


### Query Distance
### Find points NOT within distance of lines
https://gis.stackexchange.com/questions/356497/select-points-falling-outside-of-buffer-and-count
https://gis.stackexchange.com/questions/367594/get-all-geom-points-that-are-more-than-3-meters-from-the-linestring-at-big-scal
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

### Find Points which have no point within distance
https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius

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


### Find geometries close to centre of an extent
https://stackoverflow.com/questions/60218993/postgis-how-do-i-find-results-within-a-given-bounding-box-that-are-close-to-the
### Find Farthest Point from a Polygon
https://gis.stackexchange.com/questions/332073/is-there-any-function-that-can-calculate-the-maximum-minimum-distance-from-a-geo/334260#334260

### Find farthest vertex from polygon centroid
https://stackoverflow.com/questions/31497071/farthest-distance-of-a-polygon-point-from-its-centroid
### Remove Duplicate Points within given Distance
https://gis.stackexchange.com/questions/24818/remove-duplicate-points-based-on-a-specified-distance
### Find Distance and Bearing from Point to Polygon
https://gis.stackexchange.com/questions/27564/how-to-get-distance-bearing-between-a-point-and-the-nearest-part-of-a-polygon


### Use DWithin instead of Buffer
https://gis.stackexchange.com/questions/297317/st-intersects-returns-true-while-st-contains-returns-false-for-a-point-located-o

### Find closest point on boundary of a union of polygons
https://gis.stackexchange.com/questions/124158/finding-outermost-border-of-set-of-geomertries-circles-using-postgis

### Find points returned by function within elliptical area
https://gis.stackexchange.com/questions/17857/finding-points-within-elliptical-area-using-postgis

### Query Point with highest elevation along a transect through a point cloud
https://gis.stackexchange.com/questions/223154/find-highest-elevation-along-path
### Query a single point within a given distance of a road
https://gis.stackexchange.com/questions/361179/postgres-remove-duplicate-rows-returned-by-st-dwithin-query


## Query Nearest Neighbour
### Nearest Point to each point in same table
https://gis.stackexchange.com/questions/287774/nearest-neighbor

### Nearest Point to each point in different table
https://gis.stackexchange.com/questions/340192/calculating-distance-between-every-entry-in-table-a-and-nearest-record-in-table

https://gis.stackexchange.com/questions/297208/efficient-way-to-find-nearest-feature-between-huge-postgres-tables
Very thorough explanation, including difference between geom and geog
```sql
SELECT g1.gid AS gref_gid,
       g2.gid AS gnn_gid,
       g2.code_mun,
       g1.codigo_mun,
       g2.text,
       g2.via AS igcvia1
FROM u_nomen_dom As g1
JOIN LATERAL (
  SELECT gid,
         code_mun,
         text,
         via
  FROM u_nomen_via AS g
  WHERE g1.codigo_mun = g.codigo_mun    
  ORDER BY g1.geom <-> g.geom
  LIMIT 1
) AS g2
ON true;
```
https://gis.stackexchange.com/questions/136403/postgis-nearest-points-with-st-distance-knn
Lots of obsolete options, dbastons answer is best

### Snap Points to closest Point on Line
https://gis.stackexchange.com/questions/279387/automatically-snapping-points-to-closest-part-of-line
### Find Shortest Line from Points to Roads (KNN, LATERAL)
https://gis.stackexchange.com/questions/332019/distinct-st-shortestline

https://gis.stackexchange.com/questions/283794/get-barrier-edge-id

### Matching points to nearest Line Segments
https://gis.stackexchange.com/questions/296445/get-closest-road-segment-to-segmentized-linestring-points
### Using KNN with JOIN LATERAL
http://www.postgresonline.com/journal/archives/306-KNN-GIST-with-a-Lateral-twist-Coming-soon-to-a-database-near-you.html

https://gis.stackexchange.com/questions/207592/postgis-osm-faster-query-to-find-nearest-line-of-points?rq=

https://gis.stackexchange.com/questions/278357/how-to-update-with-lateral-nearest-neighbour-query
https://gis.stackexchange.com/questions/338312/find-closest-polygon-from-point-and-get-its-attributes

https://carto.com/blog/lateral-joins/
### Compute point value as average of N nearest points
https://gis.stackexchange.com/questions/349754/calculate-average-of-the-temperature-value-from-4-surrounded-points-in-postgis

#### Solution
Uses LATERAL and KNN <->
```sql
SELECT a.id,
       a.geom,
       avg(c.temp_val) temp_val
FROM tablea a
CROSS JOIN LATERAL
  (SELECT temp_val
   FROM tableb b
   ORDER BY b.geom <-> a.geom
   LIMIT 4) AS c
GROUP BY a.id,a.geom
```
### Query Nearest Neighbours having record in temporal join table 
https://gis.stackexchange.com/questions/357237/find-knn-having-reference-in-a-table

### Snapping Points to Nearest Line
https://gis.stackexchange.com/questions/365070/update-points-geometry-in-postgis-database-snapping-them-to-nearest-line
```sql
UPDATE points
  SET  geom = (
    SELECT ST_ClosestPoint(lines.geom, points.geom)
    FROM lines
    WHERE ST_DWithin(points.geom, lines.geom, 5)
    ORDER BY lines.geom <-> points.geom
    LIMIT 1
  );
```
### Find near polygons to line
https://gis.stackexchange.com/questions/377674/find-nearest-polygons-of-a-multi-line-string


## Query Geometric Shape

### Find narrow Polygons
https://gis.stackexchange.com/questions/316128/identifying-long-and-narrow-polygons-in-with-postgis

#### Solution 1 - Radius of Maximum Inscribed Circle
Use function [`ST_MaximumInscribedCircle`](https://postgis.net/docs/manual-dev/ST_MaximumInscribedCircle.html)
to compute the radius of the Maximum Inscribed Circle of each polygon, and then select using desired cut-off.
This is much faster than computing negative buffers.

#### Solution 2 - Negative Buffer
Detect thin polygons using a test: `ST_Area(ST_Buffer(geom, -10)) = 0`.
This is not very performant.

#### Solution 3 - Thinness Ratio
Use the Thinness Ratio:  `TR(Area,Perimeter) = Area * 4 * pi / (Perimter^2)`.

This is only a heuristic approximation, and it's hard to choose appropriate cut-off for the value of the ratio.

See https://gis.stackexchange.com/questions/151939/explanation-of-the-thinness-ratio-formula



## Query Invalid Geometry

### Skip invalid geometries when querying
https://gis.stackexchange.com/questions/238748/compare-only-valid-polygon-geometries-in-postgis

## Query Duplicates
### Find and Remove duplicate geometry rows
https://gis.stackexchange.com/questions/124583/delete-duplicate-geometry-in-postgis-tables


## Query with Tolerance

### Spatial Equality with Tolerance
Example: https://gis.stackexchange.com/questions/176359/tolerance-in-postgis

This post is about needing `ST_Equals` to have a tolerance to accommodate small differences caused by reprojection
https://gis.stackexchange.com/questions/141790/postgis-st-equals-false-when-st-intersection-100-of-geometry

This is about small coordinate differences defeating `ST_Equals`, and using `ST_SnapToGrid` to resolve the problem:
https://gis.stackexchange.com/questions/56617/st-equals-postgis-problems

This says that copying geometries to another database causes them to fail `ST_Equals` (not sure why copy would change the geom - perhaps done using WKT?).  Says that using buffer is too slow
https://gis.stackexchange.com/questions/213240/st-equals-not-matching-with-exact-geometry

https://gis.stackexchange.com/questions/176359/tolerance-in-postgis

### ST_ClosestPoint does not intersect Line

https://gis.stackexchange.com/questions/11510/st-closestpointline-point-does-not-intersect-line

#### Solution
Use `ST_DWithin`

### Discrepancy between GEOS predicates and PostGIS Intersects
https://gis.stackexchange.com/questions/259210/how-can-a-point-not-be-within-or-touch-but-still-intersect-a-polygon

Actually it doesn’t look like there is a discrepancy now.  But still a case where a distance tolerance might clarify things.

## Query with JOIN LATERAL

https://gis.stackexchange.com/questions/136403/postgis-nearest-points-with-st-distance-knn

https://gis.stackexchange.com/questions/291941/select-points-with-maximum-attribute-value-per-category-on-a-spatial-join-with-p

### Union of Polygons does not Cover original Polygons
https://gis.stackexchange.com/questions/376706/postgis-st-covers-doesnt-match-polygon-after-st-union
```sql
WITH data(id, pt) AS (VAlUES 
( 1, 'POINT ( 30.2833756 50.4419441) '::geometry )
,( 2, 'POINT( 30.2841370 50.4419441 ) '::geometry )
)
,poly AS (
  SELECT ST_Transform(ST_Expand( ST_Transform(ST_SetSRID( pt, 4326) , 31997), 100), 3857) AS poly
  FROM data
)
SELECT * FROM poly;
SELECT ST_Union( poly ) FROM poly;
```

