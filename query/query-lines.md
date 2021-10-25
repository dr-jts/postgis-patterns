---
parent: Querying
---

# Lines
{: .no_toc }

1. TOC
{:toc}


## Find Lines which have a given angle of incidence
<https://gis.stackexchange.com/questions/134244/query-road-shape>

#### See Also
<https://gis.stackexchange.com/questions/25126/how-to-calculate-the-angle-at-which-two-lines-intersect-in-postgis>

## Find lines which are covered by a Line using a tolerance
<https://gis.stackexchange.com/questions/408581/overlapping-lines-on-vector>

**Solution**
Use `ST_Snap`:
```sql
WITH lines AS (SELECT (ST_Dump(ST_GeomFromText(
    'GEOMETRYCOLLECTION(
        LINESTRING(-76.4631041 38.9533412, -76.45514403057082 38.962161103032216),
        LINESTRING(-76.4631041 38.9533412, -76.45349643519081 38.96398666895439),
        LINESTRING(-76.45349643519081 38.96398666895439,-76.4525346 38.9650524),
        LINESTRING(-76.45514403057082 38.962161103032216,-76.45349643519081 38.96398666895439)
    )'
))).geom AS geom)
SELECT ST_Covers(a.geom, ST_Snap(b.geom, a.geom, 0.01))
FROM lines a, lines b;
```

## Find Line Intersections
<https://gis.stackexchange.com/questions/20835/identifying-road-intersections-using-postgis>

## Find Lines which intersect N Polygons
<https://gis.stackexchange.com/questions/349994/st-intersects-with-multiple-geometries>

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
## Count number of intersections between Line Segments
<https://gis.stackexchange.com/questions/365575/counting-geometry-intersections-between-two-linestrings>

## Find Begin and End of circular sublines
<https://gis.stackexchange.com/questions/206815/seeking-algorithm-to-detect-circling-and-beginning-and-end-of-circle>

## Find Lines that form Rings
<https://gis.stackexchange.com/questions/32224/select-the-lines-that-form-a-ring-in-postgis>
Solutions
Polygonize all lines, then identify lines which intersect each polygon
Complicated recursive solution using ST_LineMerge!

## Find Longest Line Segment
<https://gis.stackexchange.com/questions/359825/get-the-maximum-distance-between-two-consecutive-points-in-a-linestring>

## Find non-monotonic Z ordinates in a LineString
<https://gis.stackexchange.com/questions/374459/postgis-validate-the-z-continuity>

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
 
## Query LineString Vertices by Measures
<https://gis.stackexchange.com/questions/340689/measuring-4d-relation-querying-within-linestring-using-postgis>

Given a Linestring wit timestamps as measure value, find segments where duration between start and end timestamps is larger than X minutes.

**Solution**
* extract vertices as geoms with `ST_DumpPoints`
* compute difference between consecutive M values using `LAG` window function
* query for the required difference

```sql
SELECT * FROM 
(select 
    ST_M(geom) - LAG( ST_M(geom)) OVER () diff, 
    path 
FROM ST_DumpPoints( 'LINESTRING M (0 0 0, 10 0 20, 12 0 40, 20 0 50, 21 0 70)')) p
WHERE diff > 10;
```

## Query Invalid Zero-Length LineStrings
<https://gis.stackexchange.com/questions/203125/st-intersects-with-degenerate-linestring?rq=1>
    
**Problem:** Zero-length LineStrings are invalid, and hence cause spatial predicates such as `ST_Intersects` to return `false`.
    
**Solution** 
    
The work-around is to run `ST_MakeValid` on the geometry.
    
```sql
SELECT ST_Intersects( 'LINESTRING (544483.525 6849134.28, 544483.525 6849134.28)', 
    'POLYGON ((543907.636214323 6848710.84802846, 543909.787417164 6849286.92923919, 544869.040437688 6849283.30837091, 544866.842236582 6848707.22673193, 543907.636214323 6848710.84802846))') AS test1,
    ST_Intersects( ST_MakeValid( 'LINESTRING (544483.525 6849134.28, 544483.525 6849134.28)' ), 
    'POLYGON ((543907.636214323 6848710.84802846, 543909.787417164 6849286.92923919, 544869.040437688 6849283.30837091, 544866.842236582 6848707.22673193, 543907.636214323 6848710.84802846))') AS test2;
```

    

