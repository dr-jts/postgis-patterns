---
parent: Overlay
---

# Union
{: .no_toc }

1. TOC
{:toc}

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
