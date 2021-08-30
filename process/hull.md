---
parent: Processing
---

# Hulls and Covers
{: .no_toc }

1. TOC
{:toc}

## Based on Lines

### Construct Bounding box of set of MULTILINESTRINGs
<https://gis.stackexchange.com/questions/115494/bounding-box-of-set-of-multilinestrings-in-postgis>
```sql
SELECT type, ST_Envelope(ST_Collect(geom))
FROM line_table AS foo
GROUP BY type;
```

### Construct polygon containing lines
<https://gis.stackexchange.com/questions/238/find-polygon-that-contains-all-linestring-records-in-postgis-table>

### Construct lines between all points of a Polygon
<https://gis.stackexchange.com/questions/58534/get-the-lines-between-all-points-of-a-polygon-in-postgis-avoid-nested-loop>

##### Solution
Rework given SQL using CROSS JOIN and a self join

## Based on Points

### Construct Regions from Points
<https://gis.stackexchange.com/questions/92913/extra-detailed-bounding-polygon-from-many-geometric-points>

### Construct regions from large sets of points (100K) tagged with region attribute.

Could use ST_ConcaveHull, but results would overlap
Perhaps ST_Voronoi would be better?  How would this work, and what are limits on size of data?

### Construct a Star Polygon from a set of Points
<https://gis.stackexchange.com/questions/349945/creating-precise-shapes-using-list-of-coordinates>

![](https://i.stack.imgur.com/vUHcU.png)

```sql
WITH pts(pt) AS (VALUES
(st_transform(st_setsrid(st_makepoint(-97.5660461, 30.4894905), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5657216, 30.4902173), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5608779, 30.4896142), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5605001, 30.491422), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5588115, 30.4911697), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5588262, 30.4910204), 4326),4269) ),
(st_transform(st_setsrid(st_makepoint(-97.5588262, 30.4910204), 4326),4269)),
(st_transform(st_setsrid(st_makepoint(-97.5585742, 30.4909966), 4326),4269)),
(st_transform(st_setsrid(st_makepoint(-97.5578045, 30.4909263), 4326),4269)),
(st_transform(st_setsrid(st_makepoint(-97.5574653, 30.4908877), 4326),4269)),
(st_transform(st_setsrid(st_makepoint(-97.5571534, 30.4908375), 4326),4269)),
(st_transform(st_setsrid(st_makepoint(-97.5560964, 30.4907427), 4326),4269))
),
centroid AS (SELECT ST_Centroid( ST_Collect(pt) ) AS centroid FROM pts),
line AS (SELECT ST_MakeLine( pt ORDER BY ST_Azimuth( centroid, pt ) ) AS geom
    FROM pts CROSS JOIN centroid),
poly AS (SELECT ST_MakePolygon( ST_AddPoint( geom, ST_StartPoint( geom ))) AS geom
    FROM line)
SELECT geom FROM poly;
```
