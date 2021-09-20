---
parent: Overlay
---

# Difference, Symmetric Difference
{: .no_toc }

1. TOC
{:toc}

## Polygon Difference

### Remove large set of polygons from a surrounding box
<https://gis.stackexchange.com/questions/330051/obtaining-the-geospatial-complement-of-a-set-of-polygons-to-a-bounding-box-in-po>

#### Issues
conventional approach is too slow to use  (Note: user never actually completed processing, so might not have encountered geometry size issues, which could also occur)

### Remove polygons from a single very large polygon
<https://gis.stackexchange.com/questions/217337/postgis-erase-logic-and-speed>.  

#### Solution
* Use `ST_Subdivide` on large polygon
* Compute difference on subdivided pieces
* Union remainders

```sql
-- Turn NJ into a large number of small tractable areas
CREATE SEQUENCE nj_square_id;
CREATE TABLE nj_squares AS
  SELECT 
    nextval('nj_square_id') AS nj_id, 
    ST_SubDivide(geom) AS geom
  FROM nj;

-- Index the squares for faster searching
CREATE INDEX nj_squares_x ON nj_squares USING GIST (geom);

-- Index parcels too in case you forgot
CREATE INDEX parcels_x ON parcels USING GIST (geom);

-- For each square, compute "bits that aren't parcels"
CREATE TABLE nj_not_parcels AS
WITH parcel_polys AS (
  SELECT nj.nj_id, ST_Union(p.geom) AS geom
  FROM nj_squares nj
  JOIN parcels p
  ON ST_Intersects(p.geom, nj.geom)
  GROUP BY nj.nj_id
)
SELECT nj_id,
  ST_Difference(nj.geom, pp.geom) AS geom
FROM parcel_polys pp 
JOIN nj_squares
USING (nj_id);
```

### Remove Polygon table from another Polygon table
<https://gis.stackexchange.com/questions/250674/postgis-st-difference-similar-to-arcgis-erase>
<https://gis.stackexchange.com/questions/187406/how-to-use-st-difference-and-st-intersection-in-case-of-multipolygons-postgis>
<https://gis.stackexchange.com/questions/90174/postgis-when-i-add-a-polygon-delete-overlapping-areas-in-other-layers>
<https://gis.stackexchange.com/questions/155597/using-st-difference-to-remove-overlapping-features>
<https://gis.stackexchange.com/questions/390281/using-postgis-to-find-the-overall-difference-between-two-large-polygon-dataset>

#### Solution
* For each target polygon, find union of all intersecting eraser polygons
* Use `LEFT JOIN` to include targets with no intersections
* Result is either difference of eraser from target, or original target (via `COALESCE`)

```sql
WITH input(geom) AS (VALUES
( 'POLYGON ((10 50, 40 50, 40 10, 10 10, 10 50))'::geometry ),
( 'POLYGON ((70 50, 70 10, 40 10, 40 50, 70 50))'::geometry ),
( 'POLYGON ((90 50, 90 10, 70 10, 70 50, 90 50))'::geometry ),
( 'POLYGON ((90 90, 90 50, 70 50, 70 90, 90 90))'::geometry )
),
eraser(geom) AS (VALUES
( 'POLYGON ((30 60, 50 60, 50 40, 30 40, 30 60))'::geometry ),
( 'POLYGON ((30 30, 50 30, 50 10, 30 10, 30 30))'::geometry ),
( 'POLYGON ((60 40, 80 40, 80 20, 60 20, 60 40))'::geometry )
)
SELECT COALESCE(
         ST_Difference(i.geom, ie.geom),
         i.geom
       ) AS geom
FROM  input AS i
LEFT JOIN LATERAL (
  SELECT ST_Union(geom) AS geom
  FROM   eraser AS e
  WHERE  ST_Intersects(i.geom, e.geom)
) AS ie ON true ;
```

Similar: Find portions of countries not covered by administrative areas.

<https://gis.stackexchange.com/questions/313039/find-what-polygons-are-not-fully-covered-by-union-of-polygons-from-another-layer>

![](https://i.stack.imgur.com/0kFJj.png)

### Remove overlaps in a Polygon table by lower-priority polygons
https://gis.stackexchange.com/questions/379300/how-to-remove-overlaps-and-keep-highest-priority-polygon

#### Solution

```sql
SELECT ST_Multi(COALESCE(
         ST_Difference(a.geom, blade.geom),
         a.geom
       )) AS geom
FROM   table1 AS a
CROSS JOIN LATERAL (
  SELECT ST_Union(geom) AS geom
  FROM   table1 AS b
  WHERE  a.prio > b.prio
) AS blade;
```

![](https://i.stack.imgur.com/W326R.png)


### Split Polygons by distance from a Polygon
<https://gis.stackexchange.com/questions/78073/separate-a-polygon-in-different-polygons-depending-of-the-distance-to-another-po>

### Cut Polygons into a Polygon table
<https://gis.stackexchange.com/questions/71461/using-st-difference-and-preserving-attributes-in-postgis>

Use Case: cut a more detailed and attributed polygon coverage dataset (`detail`) into a less detailed polygon coverage (`base`).

```sql
WITH detail_cutter AS (
  SELECT ST_Union(d.geom) AS geom, b.gid
  FROM base b JOIN detail d ON ST_Intersects(b.geom, d.geom)
  GROUP BY b.gid
),
base_rem AS (
  SELECT ST_Difference(b.geom, d.geom) AS geom, b.gid
  FROM base b JOIN detail_cutter d ON b.gid = d.gid
)
-- base remainders
SELECT 'base' AS type, b.geom, b.gid
  FROM base_rem b
UNION ALL
-- cutter polygons
SELECT 'detail' AS type, d.geom, d.gid
  FROM detail d
UNION ALL
-- uncut base polygons
SELECT 'base' AS type, b.geom, b.gid
  FROM base b 
  LEFT JOIN detail d ON ST_Intersects(b.geom,d.geom)
  WHERE d.gid is null;
```

#### Solution
* For each base polygon, union all detail polygons which intersect it (the "cutters")
* Difference the cutter union from the base polygon
* UNION ALL:
  * The base polygon remainders
  * The cutter polygons
  * The uncut base polygons

### Remove MultiPolygons from LineStrings
<https://gis.stackexchange.com/questions/239696/subtract-multipolygon-table-from-linestring-table>
<https://gis.stackexchange.com/questions/11592/difference-between-two-layers-in-postgis>

#### Solution `
```sql
SELECT COALESCE(ST_Difference(river.geom, lakes.geom), river.geom) As river_geom 
FROM river 
  LEFT JOIN lakes ON ST_Intersects(river.geom, lakes.geom);
```

#### Solution 2
<https://gis.stackexchange.com/questions/193217/st-difference-on-linestrings-and-polygons-slow-and-fails>



#### Solution
```sql
SELECT row_number() over() AS gid,
ST_CollectionExtract(ST_Multi(ST_Difference(a.geom, b.geom)), 2)::geometry(MultiLineString, 27700) as geom
FROM lines a
  JOIN LATERAL (
    SELECT ST_UNION(polygons.geom)
    FROM polygons
    WHERE ST_Intersects(a.geom,polygons.geom)
  ) AS b;
```

### Extract Block faces Lines from Parcel Polygons
<https://gis.stackexchange.com/questions/402646/get-frontage-parcels-lines-for-polygonal-parcels-in-postgis>

![](https://i.stack.imgur.com/aEXYa.jpg)

#### Solution 1
Union all polygons to get block polygons, subtract from parcel boundaries

```sql
SELECT block.id, ST_Intersection(block.geom, boundary.geom) geom
FROM block
JOIN (
SELECT ST_Boundary((ST_Dump(ST_Union(geom))).geom) geom
FROM block) boundary
ON ST_Intersects(block.geom, boundary.geom)
```
#### Solution 2
For each parcel, union all adjacent parcels and subtract from parcel boundary.

```sql
WITH parcels(id, geom) AS (VALUES
    ( 'a1', 'POLYGON ((10 10, 10 30, 30 30, 30 10, 10 10))'::geometry ),
    ( 'a2', 'POLYGON ((50 10, 30 10, 30 30, 50 30, 50 10))'::geometry ),
    ( 'a3', 'POLYGON ((10 50, 30 50, 30 30, 10 30, 10 50))'::geometry ),
    ( 'a4', 'POLYGON ((50 50, 50 30, 30 30, 30 50, 50 50))'::geometry )
)
SELECT p.id, 
       ST_Difference( ST_Boundary(p.geom), pp.geom ) AS geom
FROM parcels p
  JOIN LATERAL (SELECT ST_Collect(ST_Boundary(p2.geom)) AS geom
              FROM parcels p2 
              WHERE p.id <> p2.id AND ST_Intersects(p.geom, p2.geom) ) AS pp ON true;
```
**Advantages:**

* scales to an unlimited number of parcels
* may provide better feedback if the parcels are not noded correctly (since more of each parcel's linework is kept)

## Polygon Symmetric Difference

### Construct symmetric difference of two tables
https://gis.stackexchange.com/questions/302458/symmetrical-difference-between-two-layers


