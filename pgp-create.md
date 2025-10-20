# Create, Access, Edit
{: .no_toc }

1. TOC
{:toc}

## Geometry Creation

### Use `ST_Point` instead of `ST_MakePoint` or `ST_PointFromText`
<https://gis.stackexchange.com/questions/122247/st-makepoint-or-st-pointfromtext-to-generate-points>

<https://gis.stackexchange.com/questions/58605/which-function-for-creating-a-point-in-postgis/58630#58630>

**Solution**

`ST_Point` (and variants) is the standard function to use.  
`ST_MakePoint` is obsolete.
They are both much faster than `ST_PointFromText`.

### Collect Lines into a MultiLine in a given order
<https://gis.stackexchange.com/questions/166701/postgis-merging-linestrings-into-multilinestrings-in-a-particular-order>

Input:
![](https://i.stack.imgur.com/Oc7AG.png)

Ordering from plain `ST_Collect`:
![](https://i.stack.imgur.com/mRuXi.png)

**Solution**

Use aggregate `ORDER BY`:

```sql
SELECT ST_Collect(geom ORDER BY seq_num)) FROM lines;
```

### Create Line Segments from ordered Point records
<https://gis.stackexchange.com/questions/411386/how-to-draw-lines-from-point-to-point-in-postgis-postgresql>

**Solution**
Use `LEAD` window function:
```sql
WITH src (id, dtt, geom) AS (VALUES 
    (1,1,'POINT (1 1)'),
    (1,2,'POINT (1 2)'),
    (1,3,'POINT (1 3)'),
    (2,1,'POINT (2 1)'),
    (2,2,'POINT (2 2)'),
    (2,3,'POINT (1 3)'))
SELECT id, dtt, ST_AsText(
        ST_MakeLine(geom, LEAD(geom) OVER(PARTITION BY id ORDER BY dtt))) AS geom
FROM src;
```

## Geometry Access

### Extract shells and holes from MultiPolygons
<https://gis.stackexchange.com/questions/396367/postgis-find-outers-and-inners-inside-multipolygon-geometries>

**Solution**

A solution using:

* `generate_series` to extract the polygons (required as input to `ST_DumpRings`)
* SQL aggregate `FILTER` clauses to separate the shells and holes from the dumped rings

```sql
WITH multipoly(geom) AS (VALUES
('MULTIPOLYGON (((10 10, 10 90, 70 90, 10 10), (20 80, 40 70, 40 80, 20 80), (20 70, 40 60, 20 40, 20 70)), ((50 30, 80 60, 80 30, 50 30)))'::geometry)
),
rings AS (
  SELECT (r.dumped).geom AS geom, 
         ((r.dumped).path)[1] AS loc
    FROM (SELECT ST_DumpRings( 
                     ST_GeometryN(geom, 
                                  generate_series(1, 
                                            ST_NumGeometries( geom )))) AS dumped 
            FROM multipoly) AS r
)
SELECT  ST_Collect( geom ) FILTER (WHERE loc = 0) AS shells,
        ST_Collect( geom ) FILTER (WHERE loc > 0) AS holes
FROM rings;
```

## Geometry Editing

### Remove Holes from Polygons
<https://gis.stackexchange.com/questions/278154/polygons-have-holes-after-pgr-pointsaspolygon>
```sql
SELECT CASE
         WHEN ST_NRings(geom) > 1
         THEN ST_MakePolygon(ST_ExteriorRing(geom))
         ELSE geom
       END AS geom
FROM polys;
```

### Remove Holes from MultiPolygons
<https://gis.stackexchange.com/questions/348943/simplifying-a-multipolygon-into-one-polygon-respecting-its-outer-boundaries>

**Solution**
<https://gis.stackexchange.com/a/349016/14766>

* Use `ST_Dump` to explode the MultiPolygon into separate Polygons
* use `ST_MakePolygon(ST_ExteriorRing(poly))` to remove the holes from each element Polygon
* Use `ST_Collect` to recombine the hole-free elements 

```sql
WITH data AS (
  SELECT 'MULTIPOLYGON (((90 240, 260 240, 260 100, 90 100, 90 240), (130 200, 200 200, 200 140, 130 140, 130 200)), ((290 240, 380 240, 380 170, 290 170, 290 240), (324 216, 360 216, 360 180, 324 180, 324 216)), ((310 140, 375 140, 375 91, 310 91, 310 140)))'::geometry AS geom
),
polys AS (
  SELECT (ST_Dump( geom )).geom FROM data
),
polynoholes AS (
  SELECT ST_Collect( ST_MakePolygon( ST_ExteriorRing( geom ))) FROM polys
)
SELECT * FROM polynoholes
```

**Similar**
<https://gis.stackexchange.com/questions/291374/cut-out-polygons-that-at-least-partially-fall-in-another-polygon>

### Remove Small Holes from MultiPolygons
<https://gis.stackexchange.com/questions/431664/delete-small-holes-in-polygons-with-postgis/431682#431682>

Remove holes below a given area from a table of MultiPolygons.

**Solution**
In this example the size limit is 100.  Note that it works for Polygons as well.

```sql
WITH data(id, geom) AS (VALUES
   (1, 'MULTIPOLYGON (((100 100, 100 0, 0 0, 0 100, 100 100), (10 10, 10 70, 60 10, 10 10), (30 90, 90 90, 90 30, 30 90), (20 80, 10 80, 10 90, 20 80), (90 10, 80 10, 80 20, 90 10)), ((0 170, 100 170, 100 120, 0 120, 0 170), (10 130, 10 140, 20 130, 10 130)))'::geometry)
  ,(2, 'MULTIPOLYGON (((200 100, 300 100, 300 0, 200 0, 200 100), (210 10, 210 70, 260 10, 210 10), (280 80, 280 90, 290 80, 280 80)), ((200 160, 260 160, 260 120, 200 120, 200 160)))'::geometry)
  ,(3, 'POLYGON ((110 90, 190 90, 190 10, 110 10, 110 90), (120 20, 120 80, 180 20, 120 20), (170 70, 170 80, 180 70, 170 70))'::geometry)
)
SELECT id, ST_Collect(
    ARRAY( SELECT ST_MakePolygon(
                    ST_ExteriorRing(geom),
                    ARRAY( SELECT ST_ExteriorRing( rings.geom )
                      FROM ST_DumpRings(geom) AS rings
                      WHERE rings.path[1] > 0 
                      AND ST_Area( rings.geom ) >= 100 )
                  )
            FROM ST_Dump(geom) AS poly )
    ) AS geom
FROM data;
```
As a function:
```sql
CREATE OR REPLACE FUNCTION ST_RemoveHolesByArea(
    geom GEOMETRY,
    area real)
RETURNS GEOMETRY AS
$BODY$
WITH
   tbla AS (SELECT ST_Dump(geom))
  SELECT ST_Collect(ARRAY(SELECT ST_MakePolygon(ST_ExteriorRing(geom),
            ARRAY(SELECT ST_ExteriorRing(rings.geom) FROM ST_DumpRings(geom) AS rings
      WHERE rings.path[1]>0 AND ST_Area(rings.geom)>=area))
      FROM ST_Dump(geom))) AS geom FROM tbla;
$BODY$
LANGUAGE SQL;
```
