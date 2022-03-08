---
parent: Lines and Networks
---

# Line Processing
{: .no_toc }

1. TOC
{:toc}

## Angles and Distance

### Compute Angle at which Two Lines Intersect
<https://gis.stackexchange.com/questions/25126/how-to-calculate-the-angle-at-which-two-lines-intersect-in-postgis>

### Compute Azimuth at a Point on a Line
<https://gis.stackexchange.com/questions/178687/rotate-point-along-line-layer>

Solution
Would be nice to have a ST_SegmentIndex function to get the index of the segment nearest to a point.  Then this becomes simple.

### Compute Perpendicular Distance to a Baseline (AKA “Width” of a curve)
<https://gis.stackexchange.com/questions/54575/how-to-calculate-the-depth-of-a-linestring-using-postgis>

### Measure Length of every LineString segment
<https://gis.stackexchange.com/questions/239576/measure-length-of-each-segment-for-a-polygon-in-postgis>
Solution
Use CROSS JOIN LATERAL with generate_series and ST_MakeLine, ST_Length
ST_DumpSegments would make this much easier!

## Extracting Points

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

### Split Lines by a set of Points
<https://gis.stackexchange.com/questions/112282/splitting-lines-into-non-overlapping-subsets-based-on-points-using-postgis>

**Solutions**
* Use `ST_Split`
* Use `ST_Snap`

## Interpolating

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


## Extrapolating

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
SELECT ST_MakeLine( ST_Translate(a, sin(az1) * len, cos(az1) * len),
                    ST_Translate(b, sin(az2) * len, cos(az2) * len))
  FROM (
    SELECT a, b, 
           ST_Azimuth(a, b) AS az1, ST_Azimuth(b, a) AS az2, 
           ST_Distance(a, b) + 1 AS len
      FROM (
        SELECT ST_StartPoint(geom) AS a, ST_EndPoint(geom) AS b
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

## Merging

### Merge lines that touch at endpoints
<https://gis.stackexchange.com/questions/177177/finding-and-merging-lines-that-touch-in-postgis>

Solution given uses ST_ClusterWithin, which is clever.  Can be improved slightly however (e.g. can use ST_Boundary to get endpoints?).  Would be much nicer if ST_ClusterWithin was a window function.  

Could also use a recursive query to do a transitive closure of the “touches at endpoints” condition.  This would be a nice example, and would scale better.

Can also use ST_LineMerge to do this very simply (posted).

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

## Inserting Vertices

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

