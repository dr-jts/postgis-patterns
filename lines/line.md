---
parent: Lines and Networks
---

# Line Processing
{: .no_toc }

1. TOC
{:toc}

## Angles

### Compute Angle at which Two Lines Intersect
<https://gis.stackexchange.com/questions/25126/how-to-calculate-the-angle-at-which-two-lines-intersect-in-postgis>

### Compute Azimuth at a Point on a Line
<https://gis.stackexchange.com/questions/178687/rotate-point-along-line-layer>

Solution
Would be nice to have a ST_SegmentIndex function to get the index of the segment nearest to a point.  Then this becomes simple.

### Create Line Segment with specified angle

<https://gis.stackexchange.com/a/76066/14766>

## Distance

### Compute Perpendicular Distance to a Baseline (AKA “Width” of a curve)
<https://gis.stackexchange.com/questions/54575/how-to-calculate-the-depth-of-a-linestring-using-postgis>

### Measure Length of every LineString segment
<https://gis.stackexchange.com/questions/239576/measure-length-of-each-segment-for-a-polygon-in-postgis>
Solution
Use CROSS JOIN LATERAL with generate_series and ST_MakeLine, ST_Length
ST_DumpSegments would make this much easier!

## Line Metrics

### Sinuosity of a Line
<https://gis.stackexchange.com/questions/33041/calculating-path-sinuosity-in-postgis>


## Inserting Vertices into Lines

### Find Segment of Line Closest to Point to allow Point Insertion
<https://gis.stackexchange.com/questions/368479/finding-line-segment-of-point-on-linestring-using-postgis>

Currently requires iteration.
Would be nice if the Linear Referencing functions could return segment index.
See <https://trac.osgeo.org/postgis/ticket/892>

```sql
CREATE OR REPLACE FUNCTION ST_LineLocateN( line geometry, pt geometry )
RETURNS integer
AS $$
    SELECT i FROM (
    SELECT i, ST_Distance(
        ST_MakeLine( ST_PointN( line, s.i ), ST_PointN( line, s.i+1 ) ),
        pt) AS dist
      FROM generate_series(1, ST_NumPoints( line )-1) AS s(i)
      ORDER BY dist
    ) AS t LIMIT 1;
$$
LANGUAGE sql STABLE STRICT;
```
### Insert LineString Vertices at Closest Point(s)
<https://gis.stackexchange.com/questions/40622/how-to-add-vertices-to-existing-linestrings>
<https://gis.stackexchange.com/questions/370488/find-closest-index-in-line-string-to-insert-new-vertex-using-postgis>
<https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis>

ST_Snap does this nicely
```sql
SELECT ST_AsText( ST_Snap('LINESTRING (0 0, 9 9, 20 20)',
  'MULTIPOINT( (1 1.1), (12 11.9) )', 0.2));
```

### Add intersection points between sets of Lines (AKA Noding)
<https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis>
Problem
Add nodes into a road network from access side roads

Solutions
One post recommends simply unioning (overlaying) all the linework.  This was accepted, but has obvious problems:
Hard to extract just the road network lines
If side road falls slightly short will not create a node

## Extracting Vertices from Lines

### Select every Nth point from a LineString
<https://stackoverflow.com/questions/60319473/postgis-how-do-i-select-every-second-point-from-linestring>

### Construct evenly-spaced points along a polygon boundary
<https://gis.stackexchange.com/questions/360199/get-list-of-equidistant-points-on-polygon-border-postgis>

![](https://i.stack.imgur.com/XFfYE.png)

Note that the point count needs to be substituted in two places in the following query:
```sql
WITH shell AS (
  SELECT ST_ExteriorRing('POLYGON ((5 5, 10 15, 17 10, 5 5))') geom
)
SELECT i, ST_LineInterpolatePoint(geom, (i-1.0)/40) pt
    FROM shell
    JOIN generate_series (1, 40) AS step(i) ON true;
```

## Extracting Segments from Lines

### Extract Segments from LineStrings
<http://blog.cleverelephant.ca/2015/02/breaking-linestring-into-segments.html>

#### Using `ST_DumpSegents`
In PostGIS 3.2 this can be done with `ST_DumpSegments`:
```sql
WITH data(geom) AS (VALUES
  ('LINESTRING (1 1, 3 1, 5 2, 7 1, 9 1, 9 3)'::geometry),
  ('LINESTRING (1 5, 5 6, 9 5)'::geometry)
)
SELECT ST_AsText( (ST_DumpSegments( data.geom )).geom ) AS seg
  FROM data;
```

#### Using LATERAL JOIN with `generate_series`
This can be done in SQL using an implicit `LATERAL JOIN` against two `generate_series` calls 
on the number of line elements and the number of vertices in each line:
```sql
WITH data(id, geom) AS (VALUES
  (1, 'LINESTRING (1 1, 2 2, 3 3, 4 4)'::geometry),
  (2, 'LINESTRING (0 1, 0 2, 0 3, 0 4)'::geometry)
)
SELECT id, ST_AsText( ST_MakeLine(ST_PointN(geom, i-1), ST_PointN(geom, i))) AS seg
  FROM data 
       CROSS JOIN generate_series(2, ST_NumPoints(data.geom)) AS i;
```

#### Using `ST_Dump` and `LEAD` window function

```sql
WITH data(id, geom) AS (VALUES
  (1, 'LINESTRING (1 1, 2 2, 3 3, 4 4)'::geometry),
  (2, 'LINESTRING (0 1, 0 2, 0 3, 0 4)'::geometry)
)
SELECT * FROM (SELECT id, ST_AsText( ST_MakeLine(dmp.geom, LEAD(dmp.geom) OVER(PARTITION BY id ORDER BY dmp.path))) AS seg
                 FROM data
                 CROSS JOIN LATERAL ST_DumpPoints(geom) AS dmp) AS t
    WHERE seg IS NOT NULL;
```

### Extract Segments from MultiLineStrings

#### Using `ST_DumpSegents`
In PostGIS 3.2 this can be done with `ST_DumpSegments`:
```sql
WITH data(geom) AS (VALUES
  ('MULTILINESTRING ((1 1, 3 1, 5 2, 7 1, 9 1, 9 3), (1 3, 3 3, 5 4, 7 3))'::geometry),
  ('MULTILINESTRING ((1 5, 5 6, 9 5), (1 6, 5 7, 9 6))'::geometry)
)
SELECT ST_AsText( (ST_DumpSegments( data.geom )).geom ) AS seg
  FROM data;
```

#### Using LATERAL JOIN with `generate_series`
This can be done in SQL using an implicit `LATERAL JOIN` against two `generate_series` calls 
on the number of line elements and the number of vertices in each line:
```sql
WITH data(geom) AS (VALUES
  ('MULTILINESTRING ((1 1, 3 1, 5 2, 7 1, 9 1, 9 3), (1 3, 3 3, 5 4, 7 3))'::geometry),
  ('MULTILINESTRING ((1 5, 5 6, 9 5), (1 6, 5 7, 9 6))'::geometry)
)
SELECT ST_MakeLine(ST_PointN(line, j-1), ST_PointN(line, j)) AS seg
  FROM (SELECT ST_GeometryN(geom, i) AS line
          FROM data, 
               generate_series(1, ST_NumGeometries(data.geom)) AS i) AS t,
               generate_series(2, ST_NumPoints(t.line)) AS j;
```


## Interpolating Lines

### Construct Line substring containing a set of Points
<https://gis.stackexchange.com/questions/408337/get-the-subline-of-a-linestring-consisting-of-all-intersecting-points-without-kn>

![](https://i.stack.imgur.com/LtZM0.png)

**Solution 1 - Extract subline with new endpoints**
* use `ST_LineLocatePoint()` on each intersection point to get the fraction of the point on the line
* use `ST_LineSubstring(my_line.geom, min(fraction), max(fraction))` with a GROUP BY on the line id to get subline

**Solution 2 - Extract subline with all points added**

Comment: this probably only works for a single line and using all points in a table.

```sql
SELECT  id, ST_Makeline(geom) AS geom
FROM   (
    SELECT  lns.id, 
            ST_LineSubstring(
                ln.geom,
                ST_LineLocatePoint(ln.geom, pts.geom),
                ST_LineLocatePoint(ln.geom, LEAD(pts.geom) OVER(ORDER BY pts.id))
            ) AS geom
    FROM    <line> AS ln
    CROSS JOIN
            <points> AS pts
) q
WHERE   geom IS NOT NULL
GROUP BY id;
```


## Extrapolating Lines

### Extrapolate a Line
<https://gis.stackexchange.com/questions/33055/extrapolating-a-line-in-postgis>

Also: <https://gis.stackexchange.com/questions/367486/how-to-do-small-expansion-into-linestring-ends>

![](https://i.stack.imgur.com/FTMi3.png)

### Extending a straight Line
<https://gis.stackexchange.com/questions/104439/how-to-extend-a-straight-line-in-postgis>

Extend a line between two points by a given amount (1 m) at both ends.

**Solution**
```sql
WITH data AS ( SELECT ST_MakeLine(ST_Point(1,2), ST_Point(3,4)) AS geom )
SELECT ST_AsText( ST_MakeLine(  
                    ST_Translate(b, sin(azBA) * len, cos(azBA) * len),
                    ST_Translate(a, sin(azAB) * len, cos(azAB) * len)) )
  FROM (
    SELECT a, b, 
           ST_Azimuth(a, b) AS azAB, ST_Azimuth(b, a) AS azBA, 
           ST_Distance(a, b) + len_extend AS len
    FROM (
        SELECT ST_StartPoint(geom) AS a, ST_EndPoint(geom) AS b, 1.0 as len_extend
        FROM data
        ) AS sub
    ) AS sub2;
```

### PostGIS Ideas
`ST_LineSubstring` could be enhanced to allow fractions outside [0,1].  
Or make a new function `ST_LineExtend` (which should also handle shortening the line).

### Extend a LineString to the boundary of a polygon
<https://gis.stackexchange.com/questions/345463/how-can-i-extend-a-linestring-to-the-edge-of-an-enclosing-polygon-in-postgis>

![](https://i.stack.imgur.com/zvKKx.png)

**PostGIS Idea**
Create a new function `ST_LineExtract(line, index1, index2)` to extract a portion of a LineString between two vertex indices

### Extrapolate a Line to opposite side of Polygon
<https://gis.stackexchange.com/questions/376274/finding-opposite-side-of-a-polygon>

Given a point in a polygon and a point outside the polygon, find the point on the opposite side of the polygon lying on the line between them.

![](https://i.stack.imgur.com/WyYuO.png)

**Solution**


```sql
WITH
radius AS (SELECT ST_MakeLine( pt.geom, ST_Centroid(poly.geom) ) AS geom FROM pnt pt, polygon poly),
asilen AS (SELECT ST_Azimuth(ST_StartPoint(geom), ST_EndPoint(geom)) AS azimuth, 
                ST_Distance(ST_StartPoint(geom), ST_EndPoint(geom)) + 0.00001 AS length FROM radius),
tblc AS (SELECT ST_MakeLine(a.geom, ST_Translate(a.geom, sin(azimuth)*length, cos(azimuth)*length)) geom FROM radius a, asilen b),
tbld AS (SELECT ST_Intersection(a.geom, ST_ExteriorRing(b.geom)) geom FROM tblc a JOIN polygon b ON ST_Intersects(a.geom, b.geom)),
all_pts AS (SELECT (ST_Dump(geom)).geom geom FROM tbld)
SELECT (all_pts.geom) geom, ST_Distance(all_pts.geom, radius.geom) dist 
  FROM all_pts, radius 
  ORDER BY dist DESC LIMIT 1;
```

### Find Intersection point of disjoint Lines
<https://gis.stackexchange.com/questions/424682/intersection-of-lines-passing-from-two-line-segments-in-postgis>

![](https://i.stack.imgur.com/DEUn7.png)

**Solution**
Extend both lines so that they intersect, then compute intersection point.

```sql
WITH segments AS ( 
    SELECT ST_StartPoint('LINESTRING (4.505476754241158 51.92221504789901, 4.505379267847784 51.92221833721103)') AS s1_a, 
           ST_EndPoint(  'LINESTRING (4.505476754241158 51.92221504789901, 4.505379267847784 51.92221833721103)') AS s1_b,
           
           ST_StartPoint('LINESTRING (4.50554487780521 51.922119943633575, 4.504656820078167 51.92231795217855)') AS s2_a, 
           ST_EndPoint(. 'LINESTRING (4.50554487780521 51.922119943633575, 4.504656820078167 51.92231795217855)') AS s2_b
)
,azimuths AS ( SELECT *,
    ST_Azimuth(s1_a, s1_b) AS s1_az1,
    ST_Azimuth(s1_b, s1_a) AS s1_az2,    1 AS s1_len,
    ST_Azimuth(s2_a, s2_b) AS s2_az1,
    ST_Azimuth(s2_b, s2_a) AS s2_az2,    1 AS s2_len 
  FROM segments
)
SELECT ST_Intersection(
  ST_MakeLine( ST_Translate(s1_b, sin(s1_az1) * s1_len, cos(s1_az1) * s1_len), 
               ST_Translate(s1_a, sin(s1_az2) * s1_len, cos(s1_az2) * s1_len) ),
  ST_MakeLine( ST_Translate(s2_b, sin(s2_az1) * s2_len, cos(s2_az1) * s2_len), 
               ST_Translate(s2_a, sin(s2_az2) * s2_len, cos(s2_az2) * s2_len) )
)
FROM azimuths;
```

## Splitting Lines

### Remove Longest Segment from a LineString
<https://gis.stackexchange.com/questions/372110/postgis-removing-the-longest-segment-of-a-linestring-and-rejoining-segments>

Solution (part)
Remove longest segment, splitting linestring into two parts if needed.

Useful patterns in this code:

JOIN LATERAL generate_series to extract the line segments
array slicing to extract a subline containing a section of the original line

It would be clearer if parts of this SQL were wrapped in functions (e.g. an `ST_LineSlice` function, and a `ST_DumpSegments` function - which should become part of PostGIS).
```sql
WITH data(id, geom) AS (VALUES
    ( 1, 'LINESTRING (0 0, 1 1, 2.1 2, 3 3, 4 4)'::geometry )
),
longest AS (SELECT i AS iLongest, geom,
    ST_Distance(  ST_PointN( data.geom, s.i ),
                  ST_PointN( data.geom, s.i+1 ) ) AS dist
   FROM data JOIN LATERAL (
        SELECT i FROM generate_series(1, ST_NumPoints( data.geom )-1) AS gs(i)
     ) AS s(i) ON true
   ORDER BY dist LIMIT 1
)
SELECT 
  CASE WHEN iLongest > 2 THEN ST_AsText( ST_MakeLine(
    (ARRAY( SELECT (ST_DumpPoints(geom)).geom FROM longest))[1 : iLongest - 1]
  )) ELSE null END AS line1,
  CASE WHEN iLongest < ST_NumPoints(geom) - 1 THEN ST_AsText( ST_MakeLine(
    (ARRAY( SELECT (ST_DumpPoints(geom)).geom FROM longest))[iLongest + 1: ST_NumPoints(geom)]
  )) ELSE null END AS line2
FROM longest;
```

### Split Lines into sections of given length
<https://gis.stackexchange.com/questions/97990/break-line-into-100m-segments/334305#334305>

Modern solution using LATERAL:
```sql
WITH data AS (
    SELECT * FROM (VALUES
        ( 'A', 'LINESTRING( 0 0, 200 0)'::geometry ),
        ( 'B', 'LINESTRING( 0 100, 350 100)'::geometry ),
        ( 'C', 'LINESTRING( 0 200, 50 200)'::geometry )
    ) AS t(id, geom)
)
SELECT id, i, ST_AsText( ST_LineSubstring( d.geom, substart, LEAST(subend, 1) )) AS geom
FROM (SELECT id, geom, ST_Length(geom) len, 100 sublen FROM data) AS d
CROSS JOIN LATERAL (
    SELECT i,  
           (sublen * i)/len AS substart,
           (sublen * (i+1)) / len AS subend
    FROM generate_series(0, floor( d.len / sublen )::integer ) AS t(i)
    -- skip last i if line length is exact multiple of sublen
    WHERE (sublen * i)/len <> 1.0  
    ) AS d2;
```

**See also** 
<https://gis.stackexchange.com/questions/346196/split-a-linestring-by-distance-every-x-meters-using-postgis>

<https://gis.stackexchange.com/questions/338128/postgis-points-along-a-line-arent-actually-falling-on-the-line>
This one contains a utility function to segment a line by **length**, by using `ST_LineSubstring`.  Possible candidate for inclusion?

<https://gis.stackexchange.com/questions/360670/how-to-break-a-linestring-in-n-parts-in-postgis>

### Split Lines by Points
<https://gis.stackexchange.com/questions/441177/st-split-not-split-line-by-point>
<https://gis.stackexchange.com/questions/112282/splitting-lines-into-non-overlapping-subsets-based-on-points-using-postgis>

**Solution 1 - `ST_Snap` and `ST_Split`**
* Use `ST_Snap` to snap line to point(s) to ensure points are in line
* Use `ST_Split` to split line at points

```sql
WITH data AS (SELECT
  'LINESTRING(0 0, 100 100)'::geometry AS line,
  'POINT(51 50)':: geometry AS point
)
SELECT ST_AsText( ST_Split( ST_Snap(line, point, 1), point)) AS split
       FROM data;
```

**Solution 2 - `ST_LineLocatePoint` and `ST_LineSubstring`**
```sql
WITH
  data(_input, _blade) AS (
    VALUES (
      'SRID=3857;LINESTRING(6050668.141401841 3747562.695792065, 6050847.281693009 3748099.9265132365, 6051307.630580775 3747871.7845201474)'::GEOMETRY,
      'SRID=3857;POINT(6050714.928364518 3747735.4091125955)'::GEOMETRY
    )
  )
SELECT
  ST_LineSubstring(_input, 0, ST_LineLocatePoint(_input, _blade)) AS part1,
  ST_LineSubstring(_input, ST_LineLocatePoint(_input, _blade), 1) AS part2
  FROM data;
```

## Merging Lines

### Merge lines that touch at endpoints
<https://gis.stackexchange.com/questions/177177/finding-and-merging-lines-that-touch-in-postgis>

![](https://i.stack.imgur.com/dCCQZ.jpg)

Solution given uses `ST_ClusterWithin`.  Can be improved slightly (e.g. can use `ST_Boundary` to get endpoints?).  Would be nicer if `ST_ClusterWithin` was a window function.  

Could also use a recursive query to do a transitive closure of the “touches at endpoints” condition.  This would be a nice example, and would scale better.

Can also use `ST_LineMerge` to do this:

```sql
WITH data(geom) AS (VALUES
        ('LINESTRING (0 0, 1 1)'),
        ('LINESTRING (2 2, 1 1)'),
        ('LINESTRING (7 3, 0 0)'),
        ('LINESTRING (2 4, 2 3)'),
        ('LINESTRING (3 8, 1 5)'),
        ('LINESTRING (1 5, 2 5)'),
        ('LINESTRING (7 3, 0 7)')
),
merged AS (
    SELECT (ST_Dump(ST_LineMerge(ST_Collect(geom)))).geom
    FROM data
)
SELECT row_number() OVER () AS cid,
      geom
FROM  merged;
```

#### See Also

<https://gis.stackexchange.com/questions/360795/merge-linestring-that-intersects-without-making-them-multilinestring>
```sql
WITH data(geom) AS (VALUES
( 'LINESTRING (50 50, 150 100, 250 75)'::geometry )
,( 'LINESTRING (250 75, 200 0, 130 30, 100 150)'::geometry )
,( 'LINESTRING (100 150, 130 170, 220 190, 290 190)'::geometry )
)
SELECT ST_AsText(ST_LineMerge(ST_Collect(geom))) AS line 
FROM data;
```
### Merge lines that touch at endpoints 2
<https://gis.stackexchange.com/questions/16698/join-intersecting-lines-with-postgis>

**Solution**
<https://gis.stackexchange.com/a/80105/14766>

### Connect Lines that do not touch
<https://gis.stackexchange.com/questions/332780/merging-lines-that-dont-touch-in-postgis>

![](https://i.stack.imgur.com/ng4fX.png)

**Solution**
No built-in function to do this, but post provides a custom function.

```sql
CREATE OR REPLACE FUNCTION public.st_mergecloselines(
    geometry, geometry)
    RETURNS geometry
    LANGUAGE 'sql'
-- This function merge two lines that don't within (dont touching) returing a single multiline. By Emilson Ribeiro Neto - emilsonribeiro@hotmail.com
AS $BODY$

select ST_LineMerge(st_union(st_union($1,       
    (case
            WHEN (st_distanceSphere(ST_StartPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_StartPoint($1),ST_EndPoint($2))
                and (st_distanceSphere(ST_StartPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_EndPoint($1),ST_StartPoint($2)))
                and (st_distanceSphere(ST_StartPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_EndPoint($1),ST_EndPoint($2)))
              ) THEN st_makeLine(ST_StartPoint($1),ST_StartPoint($2))

            WHEN   (st_distanceSphere(ST_EndPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_StartPoint($1),ST_EndPoint($2))
                and (st_distanceSphere(ST_EndPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_StartPoint($1),ST_StartPoint($2)))
                and (st_distanceSphere(ST_EndPoint($1),ST_StartPoint($2)) < st_distanceSphere(ST_EndPoint($1),ST_EndPoint($2)))
              ) THEN st_makeLine(ST_EndPoint($1),ST_StartPoint($2)) 

            WHEN   (st_distanceSphere(ST_StartPoint($1),ST_EndPoint($2)) < st_distanceSphere(ST_StartPoint($1),ST_StartPoint($2))
                and (st_distanceSphere(ST_StartPoint($1),ST_EndPoint($2)) < st_distanceSphere(ST_EndPoint($1),ST_StartPoint($2)))
                and (st_distanceSphere(ST_StartPoint($1),ST_EndPoint($2)) < st_distanceSphere(ST_EndPoint($1),ST_EndPoint($2)))
              ) THEN st_makeLine(ST_StartPoint($1),ST_EndPoint($2))
            else  st_makeLine(ST_EndPoint($1),ST_EndPoint($2))
            end)),$2))

$BODY$ 
```
