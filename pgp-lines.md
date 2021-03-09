# Lines and Networks
{: .no_toc }

1. TOC
{:toc}

## Linear Referencing/Line Handling

### Select every Nth point from a LineString
https://stackoverflow.com/questions/60319473/postgis-how-do-i-select-every-second-point-from-linestring

### Extrapolate Lines
https://gis.stackexchange.com/questions/33055/extrapolating-a-line-in-postgis

ST_LineInterpolatePoint should be enhanced to allow fractions outside [0,1].

### Extend a LineString to the boundary of a polygon
https://gis.stackexchange.com/questions/345463/how-can-i-extend-a-linestring-to-the-edge-of-an-enclosing-polygon-in-postgis

Ideas
A function ST_LineExtract(line, index1, index2) to extract a portion of a LineString between two indices

### Compute Angle at which Two Lines Intersect
https://gis.stackexchange.com/questions/25126/how-to-calculate-the-angle-at-which-two-lines-intersect-in-postgis

### Compute Azimuth at a Point on a Line
https://gis.stackexchange.com/questions/178687/rotate-point-along-line-layer

Solution
Would be nice to have a ST_SegmentIndex function to get the index of the segment nearest to a point.  Then this becomes simple.

### Compute Perpendicular Distance to a Baseline (AKA “Width” of a curve)
<https://gis.stackexchange.com/questions/54575/how-to-calculate-the-depth-of-a-linestring-using-postgis>

### Measure Length of every LineString segment
https://gis.stackexchange.com/questions/239576/measure-length-of-each-segment-for-a-polygon-in-postgis
Solution
Use CROSS JOIN LATERAL with generate_series and ST_MakeLine, ST_Length
ST_DumpSegments would make this much easier!

### Find Lines that form Rings
https://gis.stackexchange.com/questions/32224/select-the-lines-that-form-a-ring-in-postgis
Solutions
Polygonize all lines, then identify lines which intersect each polygon
Complicated recursive solution using ST_LineMerge!

### Add intersection points between sets of Lines (AKA Noding)
<https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis>
Problem
Add nodes into a road network from access side roads

Solutions
One post recommends simply unioning (overlaying) all the linework.  This was accepted, but has obvious problems:
Hard to extract just the road network lines
If side road falls slightly short will not create a node

### Merge lines that touch at endpoints
<https://gis.stackexchange.com/questions/177177/finding-and-merging-lines-that-touch-in-postgis>

Solution given uses ST_ClusterWithin, which is clever.  Can be improved slightly however (e.g. can use ST_Boundary to get endpoints?).  Would be much nicer if ST_ClusterWithin was a window function.  

Could also use a recursive query to do a transitive closure of the “touches at endpoints” condition.  This would be a nice example, and would scale better.

Can also use ST_LineMerge to do this very simply (posted).

Also:

https://gis.stackexchange.com/questions/360795/merge-linestring-that-intersects-without-making-them-multilinestring
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
https://gis.stackexchange.com/questions/16698/join-intersecting-lines-with-postgis

Solution
https://gis.stackexchange.com/a/80105/14766

### Merge lines which do not form a single line
https://gis.stackexchange.com/questions/83069/cannot-st-linemerge-a-multilinestring-because-its-not-properly-ordered
Solution
Not possible with ST_LineMerge
Error is not obvious from return value though

See Also
<https://gis.stackexchange.com/questions/139227/st-linemerge-doesnt-return-linestring>

### Extract Line Segments
<https://gis.stackexchange.com/questions/174472/in-postgis-how-to-split-linestrings-into-their-individual-segments>

Need an ST_DumpSegments to do this!

### Remove Longest Segment from a LineString
https://gis.stackexchange.com/questions/372110/postgis-removing-the-longest-segment-of-a-linestring-and-rejoining-segments

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
### Split Lines into Equal-length portions
https://gis.stackexchange.com/questions/97990/break-line-into-100m-segments/334305#334305

Modern solution using LATERAL:
```sql
WITH data AS (
    SELECT * FROM (VALUES
        ( 'A', 'LINESTRING( 0 0, 200 0)'::geometry ),
        ( 'B', 'LINESTRING( 0 100, 350 100)'::geometry ),
        ( 'C', 'LINESTRING( 0 200, 50 200)'::geometry )
    ) AS t(id, geom)
)
SELECT ST_LineSubstring( d.geom, substart, 
    CASE WHEN subend > 1 THEN 1 ELSE subend END ) geom
FROM (SELECT id, geom, ST_Length(geom) len, 100 sublen FROM data) AS d
CROSS JOIN LATERAL (
    SELECT i,  
            (sublen * i)/len AS substart,
            (sublen * (i+1)) / len AS subend
        FROM generate_series(0, 
            floor( d.len / sublen )::integer ) AS t(i)
        WHERE (sublen * i)/len <> 1.0  
    ) AS d2;
```
 Need to update PG doc:  https://postgis.net/docs/ST_LineSubstring.html

See also 
https://gis.stackexchange.com/questions/346196/split-a-linestring-by-distance-every-x-meters-using-postgis

https://gis.stackexchange.com/questions/338128/postgis-points-along-a-line-arent-actually-falling-on-the-line
This one contains a nice utlity function to segment a line by length, by using ST_LineSubstring.  Possible candidate for inclusion?

https://gis.stackexchange.com/questions/360670/how-to-break-a-linestring-in-n-parts-in-postgis

### Merge Lines That Don’t Touch
https://gis.stackexchange.com/questions/332780/merging-lines-that-dont-touch-in-postgis

Solution
No builtin function to do this, but one can be created in PL/pgSQL.


### Measure/4D relation querying within linestring using PostGIS
https://gis.stackexchange.com/questions/340689/measuring-4d-relation-querying-within-linestring-using-postgis

Solution
Uses DumpPoint and windowing functions

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

### Find Segment of Line Closest to Point to allow Point Insertion
https://gis.stackexchange.com/questions/368479/finding-line-segment-of-point-on-linestring-using-postgis

Currently requires iteration.
Would be nice if the Linear Referencing functions could return segment index.
See https://trac.osgeo.org/postgis/ticket/892

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
https://gis.stackexchange.com/questions/40622/how-to-add-vertices-to-existing-linestrings
https://gis.stackexchange.com/questions/370488/find-closest-index-in-line-string-to-insert-new-vertex-using-postgis
https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis

ST_Snap does this nicely
```sql
SELECT ST_AsText( ST_Snap('LINESTRING (0 0, 9 9, 20 20)',
  'MULTIPOINT( (1 1.1), (12 11.9) )', 0.2));
```
## Graphs
### Find Shortest Path through linear network
Input Parameters: linear network MultiLineString, start point, end point

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - perhaps: construct Convex Hull, take longest diameter, find shortest path between those points

https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi

## Routing / Pathfinding

### Seaway Distances & Routes
https://www.ausvet.com.au/seaway-distances-with-postgresql/
![](https://www.ausvet.com.au/wp-content/uploads/Blog_images/seaway_1.png)

## Temporal Trajectories

### Find Coincident Paths
https://www.cybertec-postgresql.com/en/intersecting-gps-tracks-to-identify-infected-individuals/

### Remove Stationary Points
https://gis.stackexchange.com/questions/290243/remove-points-where-user-was-stationary

