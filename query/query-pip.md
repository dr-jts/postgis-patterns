---
parent: Querying
---

# Point-In-Polygon
{: .no_toc }

1. TOC
{:toc}

## Find Points contained in Polygons, keeping attributes
<https://gis.stackexchange.com/questions/354319/how-to-extract-attributes-of-polygons-at-specific-points-into-new-point-layer-in>

#### Solution
A simple query to do this is:
```sql
SELECT pt.id, poly.*
  FROM points pt
  JOIN polygons poly ON ST_Intersects(poly.geom, pt.geom);
```
Caveat: this will return multiple records if a point lies in multiple polygons. 

To ensure only a single record is returned per point, and include points which do not lie in any polygon, use:
```sql
SELECT pt.id, poly.*
  FROM points pt
  LEFT OUTER JOIN LATERAL 
    (SELECT * FROM polygons poly 
       WHERE ST_Intersects(poly.geom, pt.geom) LIMIT 1) AS poly ON true;
```
To omit points not in any polygon, use `INNER JOIN` (or just `JOIN`) instead of `LEFT OUTER JOIN`.

## Count kinds of Points in Polygons
<https://gis.stackexchange.com/questions/356976/postgis-count-number-of-points-by-name-within-distinct-polygons>
```sql
SELECT
  polyname,
  COUNT(pid) FILTER (WHERE pid='w') AS w,
  COUNT(pid) FILTER (WHERE pid='x') AS x,
  COUNT(pid) FILTER (WHERE pid='y') AS y,
  COUNT(pid) FILTER (WHERE pid='z') AS z
FROM polygons
    LEFT JOIN points ON st_intersects(points.geom, polygons.geom)
GROUP BY polyname;
```
## Optimize Point-in-Polygon query by evaluating against smaller polygons
Count lightning occurences inside countries

<https://gis.stackexchange.com/questions/83615/optimizing-st-within-query-to-count-lightning-occurrences-inside-country>

## Optimize Point-in-Polygon query by gridding polygons
Count occurences inside river polygons

<https://gis.stackexchange.com/questions/185381/optimising-a-very-large-point-in-polygon-query>

## Find smallest Polygon containing Point
<https://gis.stackexchange.com/questions/220313/point-within-a-polygon-within-another-polygon>

![](https://i.stack.imgur.com/vFDCs.png)

#### Solution
Choose containing polygon with smallest area 
```sql
SELECT DISTINCT ON (compequip.id), compequip.*, a.*
FROM compequip
LEFT JOIN a
ON ST_within(compequip.geom, a.geom)
ORDER BY compequip.id, ST_Area(a.geom)
```
## Find Points NOT in Polygons
<https://gis.stackexchange.com/questions/139880/postgis-st-within-or-st-disjoint-performance-issues>

<https://gis.stackexchange.com/questions/26156/updating-attribute-values-of-points-outside-area-using-postgis>

<https://gis.stackexchange.com/questions/313517/postgresql-postgis-spatial-join-but-keep-all-features-that-dont-intersect>

This is not PiP, but the solution of using NOT EXISTS might be applicable?

<https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table>

## Find Point in Polygon with greatest attribute
Given 2 tables:

* `obstacles` - Point layer with a column `height_m INTEGER` 
* `polyobstacles` - Polygon layer

Select the highest obstacle in each polygon. If there are several points with the same highest height a random one of those is selected.

#### Solution - JOIN LATERAL
```sql
SELECT poly.id, obs_max.*
FROM polyobstacle poly 
JOIN LATERAL (SELECT * FROM obstacles o
  WHERE ST_Contains(poly.geom, o.geom) 
 ORDER BY height_m LIMIT 1
  ) AS obs_max ON true;
```
#### Solution - DISTINCT ON
Do a spatial join between polygon and points and use `SELECT DISTINCT ON (poly.id) poly.id, o.height...`

#### Solution - ARRAY_AGG
```sql
SELECT p.id, (array_agg(o.id order by height_m))[1] AS highest_id
FROM polyobstacles p JOIN obstacles o ON ST_Contains(p.geom, o.geom)
GROUP BY p.id;
```
## Find Polygon containing Point
Basic query - with tables of address points and US census blocks, find state for each point
Discusses required indexes, and external parallelization
<https://lists.osgeo.org/pipermail/postgis-users/2020-May/044161.html>

## Count Points in Polygons with two Point tables
<https://gis.stackexchange.com/questions/377741/find-count-of-multiple-tables-that-st-intersect-a-main-table>

```sql
SELECT  ply.polyname, SUM(pnt1.cnt) AS pointtable1count, SUM(pnt2.cnt) AS pointtable2count
FROM    polytable AS ply,
        LATERAL (
          SELECT COUNT(pt.*) AS cnt
          FROM   pointtable1 AS pt
          WHERE  ST_Intersects(ply.geom, pt.geom)
        ) AS pnt1,
        LATERAL (
          SELECT COUNT(pt.*) AS cnt
          FROM   pointtable2 AS pt
          WHERE  ST_Intersects(ply.geom, pt.geom)
        ) AS pnt2
GROUP BY 1;
```
