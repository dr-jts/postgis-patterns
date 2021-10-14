---
parent: Querying
---

# Shape, Validity, Duplicates
{: .no_toc }

1. TOC
{:toc}

# Geometric Shape

## Find narrow Polygons
<https://gis.stackexchange.com/questions/316128/identifying-long-and-narrow-polygons-in-with-postgis>

![](https://i.stack.imgur.com/3ATxf.png)

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

See <https://gis.stackexchange.com/questions/151939/explanation-of-the-thinness-ratio-formula>

## Query whether Polygons are rectangular
<https://gis.stackexchange.com/questions/413944/how-to-check-which-geometry-is-rectangle>

**Solution**
If rectangles are required to have exactly 4 corners, then only polygons where ST_NPoints(geom) > 5 need to be considered.

To test quadrilaterals for rectangularity, compare the lengths of their diagonals. In a rectangle the diagonal lengths are (almost) equal. Since finite numerical precision means the lengths will rarely be exactly equal, a tolerance factor is needed To use a dimension-free tolerance the lengths can be normalized by the total length to give the **rectangularity ratio**.

A query computing the rectangularity ratio from a dataset of 2 slightly skewed rectangles and a perfect rectangle:

```sql
WITH data(id, geom) AS (VALUES
  (1, 'POLYGON((144.78116 -37.824855, 144.780843 -37.826916, 144.782018 -37.827019, 144.78232 -37.82496, 144.78116 -37.824855))')
 ,(3, 'POLYGON((153.193238 -27.682795, 153.19302 -27.68375, 153.193568 -27.683843, 153.193795 -27.682894,153.193238 -27.682795))')
 ,(4, 'POLYGON ((153.1931 -27.6828, 153.1937 -27.6828, 153.19370000000004 -27.6838, 153.1931 -27.6838, 153.1931 -27.6828))')
)
SELECT id,
     (ST_Distance( ST_PointN(ST_ExteriorRing(geom), 1), ST_PointN(ST_ExteriorRing(geom), 3))
   -  ST_Distance( ST_PointN(ST_ExteriorRing(geom), 2), ST_PointN(ST_ExteriorRing(geom), 4)))
   / (ST_Distance( ST_PointN(ST_ExteriorRing(geom), 1), ST_PointN(ST_ExteriorRing(geom), 3))
   +  ST_Distance( ST_PointN(ST_ExteriorRing(geom), 2), ST_PointN(ST_ExteriorRing(geom), 4))) AS rect_ratio
FROM data;
```

# Invalid Geometry

## Skip invalid geometries when querying
<https://gis.stackexchange.com/questions/238748/compare-only-valid-polygon-geometries-in-postgis>

# Duplicate Geometry

## Find and Remove duplicate geometry rows
<https://gis.stackexchange.com/questions/124583/delete-duplicate-geometry-in-postgis-tables>

* **Input:** table where some rows have duplicate geometry, and no identifying key
* **Ouput:** table with duplicate rows removed

```sql
SELECT geom, fld1, fld2
FROM (SELECT row_number() OVER (PARTITION BY geom) AS row_num, geom, fld1, fld2 FROM some_table) AS t
WHERE row_num = 1;
```

## Merge tables of grid cell Polygons
<https://gis.stackexchange.com/questions/412911/union-multiple-layers-reduce-rows-if-exactly-same-geometry>

* **Input:** 10 tables of grid cells, with different attributes.  Geometries are the grid cell polygons. 
   Grids may have different cell sizes, and may be overlapping.
* **Ouput:** Table with all attributes, with records for same cell merged into single record.  Missing attributes are null.

**Solution**

1. Extract unique grid cell geometries:
```sql
CREATE tmp_table AS
SELECT DISTINCT geom FROM
(
  SELECT geom from table1
  UNION
  SELECT geom from table2
... 
) 
```

2. Extract attributes via LEFT JOINs against the source tables
```sql
SELECT a.geom,t1.text1a,t1.text1b,t2.int2a,...
FROM temp_table a
LEFT JOIN table1 t1 ON a.geom = t1.geom
LEFT JOIN table2 t2 ON a.geom = t2.geom
...
```




