# Create, Access, Edit, Clean
{: .no_toc }

1. TOC
{:toc}

## Geometry Creation

### Use ST_MakePoint or ST_PointFromText
<https://gis.stackexchange.com/questions/122247/st-makepoint-or-st-pointfromtext-to-generate-points>
<https://gis.stackexchange.com/questions/58605/which-function-for-creating-a-point-in-postgis/58630#58630>

**Solution**

ST_MakePoint is much faster

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

* `generate_series` to extract the polygons (avoids having to deal with the recordset returned from `ST_Dump`)
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

### Drop Holes from Polygons
https://gis.stackexchange.com/questions/278154/polygons-have-holes-after-pgr-pointsaspolygon

### Drop Holes from MultiPolygons
https://gis.stackexchange.com/questions/348943/simplifying-a-multipolygon-into-one-polygon-respecting-its-outer-boundaries

Solution
https://gis.stackexchange.com/a/349016/14766

Similar
https://gis.stackexchange.com/questions/291374/cut-out-polygons-that-at-least-partially-fall-in-another-polygon

## Cleaning Data

### Validate Polygons
https://gis.stackexchange.com/questions/1060/what-are-the-implications-of-invalid-geometries

### Remove Ring Self-Intersections / MakeValid
https://gis.stackexchange.com/questions/15286/ring-self-intersections-in-postgis
The question and standard answer (buffer(0) are fairly mundane. But note the answer where the user uses MakeValid and keeps only the result polygons with significant area.  Might be a good option to MakeValid?

### Remove Slivers
https://gis.stackexchange.com/questions/289717/fix-slivers-holes-between-polygons-generated-after-removing-spikes

### Remove Slivers after union of non-vector clean Polygons
https://gis.stackexchange.com/questions/351912/st-union-of-set-of-polygons-result-in-polygon-with-very-small-holes

Gives outline for function `cleanPolygon` but does not provide source

### Renove Spikes
https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesSpikeRemover

https://gasparesganga.com/labs/postgis-normalize-geometry/
