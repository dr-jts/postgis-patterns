# Grouping, Clustering
{: .no_toc }

1. TOC
{:toc}

## Grouping

### Group touching Polygons
<https://gis.stackexchange.com/questions/343514/postgis-recursively-finding-intersections-of-polygons-to-determine-clusters>

![](https://i.stack.imgur.com/YkIn5.jpg)


**Solution**
Use `ST_ClusterDBSCAN`, which provides very good performance.

```sql
SELECT *,
       ST_ClusterDBSCAN(geom, 0, 1) OVER() AS clst_id
FROM   poly_tbl;
```

This problem has a similar recommended solution:
<https://gis.stackexchange.com/questions/265137/postgis-union-geometries-that-intersect-but-keep-their-original-geometries-info>

A worked example:
<https://gis.stackexchange.com/questions/366374/how-to-use-dissolve-a-subset-of-a-postgis-table-based-on-a-value-in-a-column>

An alernate solution using recursive CTE and ST_DWithin?:
<https://stackoverflow.com/questions/27081061/how-to-merge-adjactent-polygons-to-1-polygon-and-keep-min-max-data>

Similar problem in R
<https://gis.stackexchange.com/questions/254519/group-and-union-polygons-that-share-a-border-in-r>

**Issues**

* DBSCAN uses distance. This will also cluster polygons which touch only at a point, not just along an edge.  Is there a way to improve this?  Allow a different distance metric perhaps - say length of overlap?

### Grouping Intersecting Polygons
<https://gis.stackexchange.com/questions/301598/only-union-dissolve-intersecting-or-adjacent-features-to-speed-up-query>
<https://gis.stackexchange.com/questions/473030/union-of-multiple-polygons-by-value-that-st-touch>

**Solutions**
The query `data` is example data.

Using `ST_ClusterDBSCAN` with `eps => 0`:

```sql
WITH data(fid, class, geom) AS (
  SELECT ROW_NUMBER() OVER (), 
      CASE x WHEN 5 THEN 1 ELSE x END AS class,
      ST_Buffer(ST_Point(x, 1.5 * y), 0.6, 2) AS geom
    FROM        generate_series(1, 10) AS s1(y)
    CROSS JOIN  generate_series(1, 10) AS s2(x)
),
clust AS (
  SELECT ST_ClusterDBSCAN(geom, 0, 2) OVER () AS clustid, fid, class, geom 
  FROM data WHERE class IN (1, 2, 3)
)
SELECT * FROM clust WHERE clustid IS NOT NULL;
```

Using `ST_ClusterIntersectingWin`:
```sql
WITH data(fid, class, geom) AS (
  SELECT ROW_NUMBER() OVER (), 
      CASE x WHEN 5 THEN 1 ELSE x END AS class,
      ST_Buffer(ST_Point(x, 1.5 * y), 0.6, 2) AS geom
    FROM        generate_series(1, 10) AS s1(y)
    CROSS JOIN  generate_series(1, 10) AS s2(x)
),
datasel AS (
  SELECT * FROM data WHERE class IN (1, 2, 3)
),
clust AS (
  SELECT ST_ClusterIntersectingWin(geom) OVER () AS clustid, fid, class, geom FROM datasel
),
clustercnt AS (
  SELECT clustid, COUNT(*) AS cnt FROM clust GROUP BY clustid
)
SELECT fid, class, geom FROM clust c
  JOIN clustercnt cs ON c.clustid = cs.clustid
  WHERE cnt > 1;
```


### Group connected LineStrings
<https://gis.stackexchange.com/questions/94203/grouping-connected-linestrings-in-postgis>

![](https://i.stack.imgur.com/WNlxX.png)

Presents a recursive CTE approach, but ultimately recommends using `ST_ClusterDBCSAN` or `ST_ClusterIntersecting`.

### Group LineStrings via connectivity and attribute
<https://gis.stackexchange.com/questions/189091/postgis-how-to-merge-contiguous-features-sharing-same-attributes-values>

![](https://i.stack.imgur.com/SFl4X.png)

Use `ST_ClusterIntersecting`:
```sql
SELECT attr, unnest(ST_ClusterIntersecting(geom))
FROM lines
GROUP by attr;
```

### Group Polygon Coverage into similar-sized Areas
<https://gis.stackexchange.com/questions/350339/how-to-create-polygons-of-a-specific-size>

More generally: how to group adjacent polygons into sets with similar sum of a given attribute.

See Also
<https://gis.stackexchange.com/questions/123289/grouping-village-points-based-on-distance-and-population-size-using-arcgis-deskt>

Solution
Build adjacency graph and aggregate based on total, and perhaps some distance criteria?
Note: posts do not provide a PostGIS solution for this.  Not known if such a solution exists.
Would need a recursive query to do this.
How to keep clusters compact?

### GROUP BY keeping one Geometry
<https://stackoverflow.com/questions/67972764/alternative-first-function-for-geometry-type>

Using `FIRST_VALUE` window function:
```sql
SELECT DISTINCT col,
       FIRSET_VALUE(geom) OVER (PARTITION BY col ORDER BT dt)
FROM t;
```
Using `ARRAY_AGG` approach:
```sql
SELECT col,
       (ARRAY_AGG(geo ORDER BY dt))[1]
FROM t GROUP BY col
```
### Group points by attribute and compute a representative location
<https://gis.stackexchange.com/questions/269407/centroid-of-point-cluster-points>

Use `ST_Collect` with `ST_Centroid` or `ST_GeometricMedian`:
```sql
SELECT village, ST_Centroid(ST_Collect(geom)) AS geom FROM your_table GROUP BY village;
```

## Clustering
See <https://gis.stackexchange.com/questions/11567/spatial-clustering-with-postgis> 
for a variety of approaches that predate a lot of the PostGIS clustering functions.

### Grid-Based Clustering
<https://gis.stackexchange.com/questions/352562/is-it-possible-to-get-one-geometry-per-area-box-in-postgis>

Solution 1
Use `ST_SnapToGrid` to compute a cell id for each point, then bin the points based on that.  Can use aggregate function to count points in grid cell, or use DISTINCT ON as a cheesy way to pick one representative point.  Need to use representative point rather than average, for better visual results (perhaps?)

Solution 2
Generate grid of cells covering desired area, then `JOIN LATERAL` to points to aggregate.  Not sure how to select a representative point doing this though - perhaps MIN or MAX?  Requires a grid-generating function, which is coming in PostGIS 3.1

### Clustering Points using DBSCAN

Cluster a selection of points using DBSCAN, and return centroid of cluster and count of points.

<https://gis.stackexchange.com/questions/388848/clustering-and-combining-points-in-postgis>

```sql
WITH pts AS (
    SELECT (ST_DumpPoints(ST_GeneratePoints('POLYGON ((10 90, 90 90, 90 10, 10 10, 10 90))', 100, 2))).geom AS geom
)
SELECT x.cid, ST_Centroid(ST_Collect(x.geom)) geom, COUNT(cid) num_points FROM
(
    SELECT ST_ClusterDBSCAN(geom, eps := 8, minpoints := 2) over () AS cid, geom
    FROM pts
    GROUP BY(geom)
) as x
WHERE cid IS NOT NULL
GROUP BY x.cid ORDER BY cid;
```

![](https://i.stack.imgur.com/auPHl.jpg)

### Non-spatial clustering by distance
<https://stackoverflow.com/questions/49250734/sql-window-function-that-groups-values-within-selected-distance>

### Find Density Centroids Within Polygons
<https://gis.stackexchange.com/questions/187256/finding-density-centroids-within-polygons-in-postgis>


### Kernel Density
<https://gist.github.com/AbelVM/dc86f01fbda7ba24b5091a7f9b48d2ee>

### Bottom-Up Clustering Algorithm
Not sure if this is worthwhile or not.  Possibly superseded by more recent standard PostGIS clustering functions

<https://gis.stackexchange.com/questions/113545/get-a-single-cluster-from-cloud-of-points-with-specified-maximum-diameter-in-pos>

### Using ClusterWithin VS ClusterDBSCAN
<https://gis.stackexchange.com/questions/348484/clustering-points-in-postgis>

Explains how DBSCAN is a superset of `ST_ClusterWithin`, and provides simpler, more powerful SQL.


### Removing Clusters of Points
<https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius>

### Cluster with DBSCAN partitioned by polygons
<https://gis.stackexchange.com/questions/284190/python-cluster-points-with-dbscan-keeping-into-account-polygon-boundaries>

### FInd polygons which are not close to any other polygon
<https://gis.stackexchange.com/questions/312167/calculating-shortest-distance-between-polygons>

Solution
Use `ST_GeometricMedian`

### Cluster with DBSCAN partitioned by record types
<https://gis.stackexchange.com/questions/357838/how-to-cluster-points-with-st-clusterdbscan-taking-into-account-their-type-store>

### Select evenly-distributed points of unevenly-distributed set, with priority
<https://gis.stackexchange.com/questions/346412/select-evenly-distirbuted-points-of-unevenly-distributed-set>

### Construct K-Means clusters for each Polygon
<https://gis.stackexchange.com/questions/376563/cluster-points-in-each-polygon-into-n-parts>

![](https://i.stack.imgur.com/eFj0g.png)

Use the `ST_ClusterKMeans` window function with `PARTITION BY`.

`ST_ClusterKMeans` computes cluster ids 0..n for each (hierarchy of) partition key(s) used in the PARTITION BY expression.

To compute clusters for each set of points in each polygon (assuming each polygon has id `poly_id`):

```sql
WITH polys(poly_id, geom) AS (
        VALUES  (1, 'POLYGON((0 0, 0 5, 5 5, 5 0, 0 0))'::GEOMETRY),
                (2, 'POLYGON((10 10, 10 15, 15 15, 15 10, 10 10))'::GEOMETRY)
    )
SELECT  polys.poly_id,
        ST_ClusterKMeans(pts.geom, 4) OVER(PARTITION BY polys.poly_id) AS cluster_id,
        pts.geom
FROM    polys,
        LATERAL ST_Dump(ST_GeneratePoints(polys.geom, 1000, 1)) AS pts
ORDER BY 1, 2;
```
![](https://i.stack.imgur.com/3Tz4X.png)


