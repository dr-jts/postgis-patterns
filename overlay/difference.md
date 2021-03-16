
---
parent: Overlay
---

# Difference, Symmetric Difference
{: .no_toc }

1. TOC
{:toc}

## Polygon Difference

### Subtract large set of polygons from a surrounding box
https://gis.stackexchange.com/questions/330051/obtaining-the-geospatial-complement-of-a-set-of-polygons-to-a-bounding-box-in-po/333562#333562

#### Issues
conventional approach is too slow to use  (Note: user never actually completed processing, so might not have encountered geometry size issues, which could also occur)

### Subtract MultiPolygons from LineStrings
https://gis.stackexchange.com/questions/239696/subtract-multipolygon-table-from-linestring-table

https://gis.stackexchange.com/questions/11592/difference-between-two-layers-in-postgis
#### Solution
```sql
SELECT COALESCE(ST_Difference(river.geom, lakes.geom), river.geom) As river_geom 
FROM river 
  LEFT JOIN lakes ON ST_Intersects(river.geom, lakes.geom);
```

https://gis.stackexchange.com/questions/193217/st-difference-on-linestrings-and-polygons-slow-and-fails



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

### Split Polygons by distance from a Polygon
https://gis.stackexchange.com/questions/78073/separate-a-polygon-in-different-polygons-depending-of-the-distance-to-another-po

### Cut Polygons into a Polygonal coverage
https://gis.stackexchange.com/questions/71461/using-st-difference-and-preserving-attributes-in-postgis

#### Solution
For each base polygon, union all detailed polygons which intersect it
Difference the detailed union from the each base polygon
UNION ALL:
The differenced base polygons
The detailed polygons
All remaining base polygons which were not changed

### Subtract Areas from a set of Polygons
https://gis.stackexchange.com/questions/250674/postgis-st-difference-similar-to-arcgis-erase

https://gis.stackexchange.com/questions/187406/how-to-use-st-difference-and-st-intersection-in-case-of-multipolygons-postgis

https://gis.stackexchange.com/questions/90174/postgis-when-i-add-a-polygon-delete-overlapping-areas-in-other-layers

https://gis.stackexchange.com/questions/155597/using-st-difference-to-remove-overlapping-features

```sql
SELECT id, COALESCE(ST_Difference(geom, (SELECT ST_Union(b.geom) 
                                         FROM parcels b
                                         WHERE ST_Intersects(a.geom, b.geom)
                                         AND a.id != b.id)), 
                    a.geom)
FROM parcels a;
```

### Find Part of Polygons not fully contained by union of other Polygons
Find what countries are not fully covered by administrative boundaries and the geometry of part of country geometry 
where it is not covered by the administrative boundaries.

https://gis.stackexchange.com/questions/313039/find-what-polygons-are-not-fully-covered-by-union-of-polygons-from-another-layer

![](https://i.stack.imgur.com/0kFJj.png)

### Remove overlaps by lower-priority polygons
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



## Polygon Symmetric Difference

### Construct symmetric difference of two tables
https://gis.stackexchange.com/questions/302458/symmetrical-difference-between-two-layers


