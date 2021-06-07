---
parent: Querying
---

# Spatial Relationship
{: .no_toc }

1. TOC
{:toc}

## General

### Find geometries in a table which do NOT intersect another table
https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table

Use NOT EXISTS:
```sql
SELECT * FROM polygons
WHERE NOT EXISTS (SELECT 1 FROM streets WHERE ST_Intersects(polygons.geom, streets.geom))
```

### Test if two 3D geometries are equal
https://gis.stackexchange.com/questions/373978/how-to-check-two-3d-geometry-are-equal-in-postgis/373980#373980

## With Tolerance

### Spatial Equality with Tolerance
Example: <https://gis.stackexchange.com/questions/176359/tolerance-in-postgis>

This post is about needing `ST_Equals` to have a tolerance to accommodate small differences caused by reprojection
<https://gis.stackexchange.com/questions/141790/postgis-st-equals-false-when-st-intersection-100-of-geometry>

This is about small coordinate differences defeating `ST_Equals`, and using `ST_SnapToGrid` to resolve the problem:
<https://gis.stackexchange.com/questions/56617/st-equals-postgis-problems>

This says that copying geometries to another database causes them to fail `ST_Equals` (not sure why copy would change the geom - perhaps done using WKT?).  Says that using buffer is too slow
<https://gis.stackexchange.com/questions/213240/st-equals-not-matching-with-exact-geometry>

<https://gis.stackexchange.com/questions/176359/tolerance-in-postgis>

### ST_ClosestPoint does not intersect Line

<https://gis.stackexchange.com/questions/11510/st-closestpointline-point-does-not-intersect-line>

#### Solution
Use `ST_DWithin`

### Discrepancy between GEOS predicates and PostGIS Intersects
<https://gis.stackexchange.com/questions/259210/how-can-a-point-not-be-within-or-touch-but-still-intersect-a-polygon>

Actually it doesn’t look like there is a discrepancy now.  But still a case where a distance tolerance might clarify things.

## Polygon / Polygon

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

### Find Polygons with maximum overlap with another Polygon table

#### Solution with JOIN LATERAL
<https://gis.stackexchange.com/questions/287412/in-postgresql-how-to-get-the-polygon-id-that-intersects-the-most-in-case-it-i>

Example: For each mountain area, find country which has largest overlap.
```sql
SELECT a.id, a.mountain, b.country
FROM mountains a
LEFT JOIN LATERAL
  (SELECT country FROM countrytable
    WHERE ST_Intersects(countrytable.geom, a.geom)
    ORDER BY ST_Area(ST_Intersection(countrytable.geom, a.geom)) DESC NULLS LAST
    LIMIT 1
  ) b ON true;
```

#### Solution using DISTINCT ON
<https://gis.stackexchange.com/questions/41501/join-based-on-maximum-overlap-in-postgis-postgresql>

1) Calculate area of intersection for every pair of rows which intersect.
2) Order them by a.id and by intersection area when a.id are equal.
3) For each `a.id` keep the first row (which has the largest intersection area because of ordering in step 2).

```sql
SELECT DISTINCT ON (a.id)
  a.id AS a_id,
  b.id AS b_id,
  ST_Area(ST_Intersection( a.geom, b.geom )) AS intersect_area
FROM a, b
WHERE st_intersects( a.geom, b.geom)
ORDER BY a.id, ST_Area(ST_Intersection( a.geom, b.geom )) DESC
```

### Compute hierarchy of a nested Polygonal Coverage
<https://gis.stackexchange.com/questions/343100/intersecting-polygons-to-build-boundary-hierearchy-using-postgis>

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


## Polygon / Line

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

## Line / Line

### Find LineStrings with Common Segments
https://gis.stackexchange.com/questions/268147/find-linestrings-with-common-segments-in-postgis-2-3
```sql
SELECT ST_Relate('LINESTRING(0 0, 2 0)'::geometry,
                 'LINESTRING(1 0, 2 0)'::geometry,
                 '1********');
```                 

## Line / Point

### Test if Point is on a Line
https://gis.stackexchange.com/questions/11510/st-closestpointline-point-does-not-intersect-line

Also https://gis.stackexchange.com/questions/350461/find-path-containing-point


