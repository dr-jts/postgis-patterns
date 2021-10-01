---
parent: Querying
---

# Shape, Validity, Duplicates

{: .no_toc }

1. TOC
{:toc}

## Query Geometric Shape

### Find narrow Polygons
<https://gis.stackexchange.com/questions/316128/identifying-long-and-narrow-polygons-in-with-postgis>

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



## Query Invalid Geometry

### Skip invalid geometries when querying
<https://gis.stackexchange.com/questions/238748/compare-only-valid-polygon-geometries-in-postgis>

## Query Duplicates

### Find and Remove duplicate geometry rows
<https://gis.stackexchange.com/questions/124583/delete-duplicate-geometry-in-postgis-tables>

* **Input:** table where some rows have duplicate geometry, and no identifying key
* **Ouput:** table with duplicate rows removed

```sql
SELECT geom, fld1, fld2
FROM (SELECT row_number() OVER (PARTITION BY geom) AS row_num, geom, fld1, fld2 FROM some_table) AS t
WHERE row_num = 1;
```

### Merge tables of grid cell polygons
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




