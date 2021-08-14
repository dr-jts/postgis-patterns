---
parent: Querying
---

# Query Lines
{: .no_toc }

1. TOC
{:toc}


## Find Lines which have a given angle of incidence
https://gis.stackexchange.com/questions/134244/query-road-shape?noredirect=1&lq=1

## Find Line Intersections
https://gis.stackexchange.com/questions/20835/identifying-road-intersections-using-postgis

## Find Lines which intersect N Polygons
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
## Count number of intersections between Line Segments
https://gis.stackexchange.com/questions/365575/counting-geometry-intersections-between-two-linestrings

## Find Begin and End of circular sublines
https://gis.stackexchange.com/questions/206815/seeking-algorithm-to-detect-circling-and-beginning-and-end-of-circle

## Find Longest Line Segment
https://gis.stackexchange.com/questions/359825/get-the-maximum-distance-between-two-consecutive-points-in-a-linestring

## Find non-monotonic Z ordinates in a LineString
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
## Find Polygons close to Lines but not intersecting them
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
