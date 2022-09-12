---
parent: Querying
---

# Nearest-Neighbour
{: .no_toc }

1. TOC
{:toc}

### Technical Explanation
<https://blog.crunchydata.com/blog/a-deep-dive-into-postgis-nearest-neighbor-search>

`<->` Operator [manual](https://postgis.net/docs/geometry_distance_knn.html).

PostGIS workshop - [Nearest Neighbour Searching](https://postgis.net/workshops/postgis-intro/knn.html).

### Using KNN with JOIN LATERAL
<http://www.postgresonline.com/journal/archives/306-KNN-GIST-with-a-Lateral-twist-Coming-soon-to-a-database-near-you.html>

<https://gis.stackexchange.com/questions/207592/postgis-osm-faster-query-to-find-nearest-line-of-points>

<https://gis.stackexchange.com/questions/278357/how-to-update-with-lateral-nearest-neighbour-query>
<https://gis.stackexchange.com/questions/338312/find-closest-polygon-from-point-and-get-its-attributes>

<https://carto.com/blog/lateral-joins/>



## Point / Point

### Find Nearest Point to Points in same table of geodetic points
<https://gis.stackexchange.com/questions/287774/nearest-neighbor>

**Solution**

Note casts to `geography` to compute correct geodetic distance.

```sql
SELECT p1.id AS id1,
       p2.id AS id2,
       ST_Distance(p1.geom::geography, p2.geom::geography) AS dist
FROM   points AS p1
CROSS JOIN LATERAL (
  SELECT id,
         geom
  FROM   points
  WHERE  p1.id <> id
 -- AND  ST_DWithin(p1.geom::geography, geom::geography, 10000)
  ORDER BY p1.geom::geography <-> geom::geography
  LIMIT  1
) AS p2;
```

### Nearest Point to Points in different table
<https://gis.stackexchange.com/questions/340192/calculating-distance-between-every-entry-in-table-a-and-nearest-record-in-table>

<https://gis.stackexchange.com/questions/297208/efficient-way-to-find-nearest-feature-between-huge-postgres-tables>
Very thorough explanation, including difference between geom and geog
```sql
SELECT g1.gid AS gref_gid,
       g2.gid AS gnn_gid,
       g2.code_mun,
       g1.codigo_mun,
       g2.text,
       g2.via AS igcvia1
FROM u_nomen_dom As g1
JOIN LATERAL (
  SELECT gid,
         code_mun,
         text,
         via
  FROM u_nomen_via AS g
  WHERE g1.codigo_mun = g.codigo_mun    
  ORDER BY g1.geom <-> g.geom
  LIMIT 1
) AS g2
ON true;
```

<https://gis.stackexchange.com/questions/401425/postgis-closest-point-to-another-point>

<https://gis.stackexchange.com/questions/136403/postgis-nearest-points-with-st-distance-knn>
Lots of obsolete options, dbastons answer is best

## Match locations to nearest points in another set

<https://gis.stackexchange.com/questions/360282/find-all-closest-points-to-unique-point-in-postgis>
**Use Case:** Given a set of collision locations and a set of intersections, match collisions to the nearest intersection.

**Solution**

This is efficiently computed using a KNN query, iterating over each collision to find the nearest intersection.

```sql
SELECT c.*, i.id, i.geom
FROM collisions c 
CROSS JOIN LATERAL 
    (SELECT id, geom     
       FROM intersections    
       ORDER BY c.geom <-> geom 
       LIMIT 1) i;
```

### Compute point value as average of N nearest points
<https://gis.stackexchange.com/questions/349754/calculate-average-of-the-temperature-value-from-4-surrounded-points-in-postgis>

#### Solution
Use `JOIN LATERAL` and KNN `<->`:
```sql
SELECT a.id, a.geom, AVG(c.temp_val) temp_val
FROM tablea a
CROSS JOIN LATERAL
  (SELECT temp_val
   FROM tableb b
   ORDER BY b.geom <-> a.geom
   LIMIT 4) AS c
GROUP BY id, geom
```

### Find Nearest Neighbours having record in temporal join table 
<https://gis.stackexchange.com/questions/357237/find-knn-having-reference-in-a-table>

## Point / Line

### Find Shortest Line from Points to Roads (KNN, LATERAL)
<https://gis.stackexchange.com/questions/332019/distinct-st-shortestline>

<https://gis.stackexchange.com/questions/283794/get-barrier-edge-id>

<https://gis.stackexchange.com/questions/296445/get-closest-road-segment-to-segmentized-linestring-points>

### Snap Points to Nearest Point on Line
<https://gis.stackexchange.com/questions/279387/automatically-snapping-points-to-closest-part-of-line>

<https://gis.stackexchange.com/questions/365070/update-points-geometry-in-postgis-database-snapping-them-to-nearest-line>
```sql
UPDATE points
  SET  geom = (
    SELECT ST_ClosestPoint(lines.geom, points.geom)
    FROM lines
    WHERE ST_DWithin(points.geom, lines.geom, 5)
    ORDER BY lines.geom <-> points.geom
    LIMIT 1
  );
```

