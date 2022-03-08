---
parent: Overlay
---

# Union
{: .no_toc }

1. TOC
{:toc}

## Polygon Union

### Union Polygons from two tables, grouped by name
<https://gis.stackexchange.com/questions/378699/merging-two-multipolygon-tables-into-one-and-afterwards-dissolve-boundaries>

![](https://i.stack.imgur.com/AUz4x.png)

```sql
SELECT name, ST_Multi(ST_Union(geom)) AS geom
FROM ( 
  SELECT name, geom FROM table1
  UNION ALL
  SELECT name, geom FROM table2 ) q
GROUP BY name;
```

### Boundary of Coverage of Polygons
<https://gis.stackexchange.com/questions/324736/extracting-single-boundary-from-multipolygon-in-postgis>

**Solution**
Union, then extract boundary

### Union of set of geometry specified by IDs
```sql
SELECT ST_Union(geom)) 
  FROM ( SELECT geom FROM table WHERE id IN ( … ) ) as t;
```

### Union of cells grouped by ID
<https://gis.stackexchange.com/questions/288880/finding-geometry-of-cluster-from-points-collection-using-postgis>


### Union of polygons with equal or lower value
<https://gis.stackexchange.com/questions/161849/postgis-sql-request-with-aggregating-st-union>

**Solution**
Use of window functions with `PARTITION BY` and `ORDER BY`.

Not sure what happens if there are two polygons with same value though?


### Union Non-clean Polygons
<https://gis.stackexchange.com/questions/31895/joining-lots-of-small-polygons-to-form-larger-polygon-using-postgis>

![](https://i.stack.imgur.com/5P53M.png)

**Solution**
Snap Polygons to grid to help clean invalid polygons 
```sql
 SELECT ST_Union(ST_SnapToGrid(the_geom,0.0001)) 
 FROM parishes
 GROUP BY county_name;
```

## Union Edge-Adjacent Groups

### Union Edge-Adjacent Polygons
<https://gis.stackexchange.com/questions/54848/building-larger-polygons-from-smaller-ones-without-common-id-in-postgis>

![](https://i.stack.imgur.com/bTRzU.png)

**Solution**

Group the polygons via an intersects relationship and union the partitions.
This can be done in-memory using `ST_ClusterIntersecting`, or non-memory bound by using `ST_ClusterDBSCAN`.

```sql
SELECT ST_UnaryUnion( UNNEST( ST_ClusterIntersecting(geom) ) ) FROM polys;
```

### Union Edge-Adjacent Polygons, keeping attributes
Only union polygons which share an edge (not just touch).

* <https://gis.stackexchange.com/questions/24634/merging-polygons-that-intersect-by-more-than-a-specified-amount>
* <https://gis.stackexchange.com/questions/127019/merge-any-and-all-adjacent-polygons>

**Solution**
No good solution so far.  
What is needed is a function similar to `ST_ClusterIntersecting` but which does not group polygons which touch only at points.

### Union Groups of Adjacent Polygon, keeping attribution for singletons
<https://gis.stackexchange.com/questions/366374/how-to-use-dissolve-a-subset-of-a-postgis-table-based-on-a-value-in-a-column>

**Solution**
Use `ST_ClusterDBSCAN` with a zero (or very small) distance

## Union with Gap Removal

### Polygon Coverage Union with slivers removed
<https://gis.stackexchange.com/questions/71809/is-there-a-dissolve-st-union-function-that-will-close-gaps-between-features>

Solution - NOT SURE

### Polygon Coverage Union with gaps removed
<https://gis.stackexchange.com/questions/356480/is-it-possible-create-a-polygon-from-more-polygons-that-are-not-overlapped-but-c>

<https://gis.stackexchange.com/questions/316000/using-st-union-to-combine-several-polygons-to-one-multipolygon-using-postgis>

Both of these have answers recommending using a small buffer outwards and then the inverse on the result.

### Union groups of almost-adjacent polygons
<https://gis.stackexchange.com/questions/185393/what-is-the-best-way-to-merge-lots-of-small-adjacents-polygons-postgis>

![](https://i.stack.imgur.com/1RCSR.png)

**Solution** 
(lossy)
```sql
SELECT (ST_DUMP(ST_UNION(ST_SNAPTOGRID(the_geom,0.0001)))).geom, color
FROM my_poly
GROUP BY color
```

### Enlarge Polygons to Fill Boundary
<https://gis.stackexchange.com/questions/91889/adjusting-polygons-to-boundary-and-filling-holes>

### Create polygons that fill gaps
<https://gis.stackexchange.com/questions/60655/creating-polygons-from-gaps-in-postgis>

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


## Union of Large datasets

### Union of Massive Number of Point Buffers using GeoHash spatial partitioning
<https://gis.stackexchange.com/questions/31880/memory-issue-when-trying-to-buffer-union-large-dataset-using-postgis>

Union a massive number of buffers around points which have an uneven distribution (points are demographic data in the UK).
Using plain `ST_Union` runs out of memory.

![](https://i.stack.imgur.com/BFQ5w.jpg)

**Solution**

Implement a “SQL-level” **cascaded union**:
* spatially sort data based on `ST_GeoHash`
* union smaller partitions of the data (e.g. partition size = 100K)
* union the partitions together 

```sql
CREATE SEQUENCE bseq;

WITH ordered AS (
  SELECT ST_Buffer(geom, 10) AS geom
  FROM points
  ORDER BY ST_GeoHash(geom)
),
grouped AS (
  SELECT nextval('bseq') / 100000 AS id, ST_Union(geom) AS geom
  FROM ordered
  GROUP BY id
)
groupedfinal AS (
  SELECT ST_Union(geom) AS geom
  FROM grouped
)
SELECT * FROM groupedfinal;
```

### Union by Spatial Partition via Intersection
<https://gis.stackexchange.com/a/424958/14766>

If a dataset is fairly sparse, it may provide a performance and memory advantage to union
by groups of geometries partitioned by a "touches" relation.  
This can be done by using grouping the geometries via `ST_ClusterDBSCAN` with a zero or small distance,
and then unioning the groups. 

If needed the result could then be unioned once again to create a single result geometry.
In theory the partitions should be disjoint, so potentially just collecting them should be faster
and produce a valid MultiPolygon.

![](https://i.stack.imgur.com/yN31B.png)

**Solution**
```sql
SELECT ST_Union(geom) AS geom
  FROM ( SELECT geom,
           ST_ClusterDBSCAN(geom, 0, 1) OVER () AS _id
         FROM input
         GROUP BY _id);
```

### Union Large Datasets (Questions only)

These questions are looking for union of large sets of polygons.

* <https://gis.stackexchange.com/questions/78630/postgis-union-multiple-tables-big-dataset-faster-approach>
* <https://gis.stackexchange.com/questions/187728/alternative-to-st-union-st-memunion-for-merging-overlapping-polygons-using-postg>
* <https://gis.stackexchange.com/questions/1387/is-there-a-dissolve-function-in-postgis-other-than-st-union>


