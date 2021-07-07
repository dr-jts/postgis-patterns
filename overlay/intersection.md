---
parent: Overlay
---

# Intersection, Clipping
{: .no_toc }

1. TOC
{:toc}

## Polygon Intersection

### Aggregated Intersection
<https://gis.stackexchange.com/questions/271824/st-intersection-intersection-of-all-geometries-in-a-table>
<https://gis.stackexchange.com/questions/271941/looping-through-table-to-get-single-intersection-from-n2-geometries-using-postg>
<https://gis.stackexchange.com/questions/60281/query-to-find-the-intersection-coordinates-of-multiple-polyon>
<https://gis.stackexchange.com/questions/269875/aggregate-version-of-st-intersection>

#### Solutions
* Define an aggregate function (given in #2)
* Define a function to do the looping
* Use a recursive CTE (see SQL in #2)

```sql
CREATE AGGREGATE ST_IntersectionAgg (
  basetype = geometry,
  stype = geometry,
  sfunc = ST_Intersection
);
```
**Example:**
```sql
WITH data(geom) AS (VALUES
( ST_Buffer(ST_Point(0,0), 0.75) ),
( ST_Buffer(ST_Point(1,0), 0.75) ),
( ST_Buffer(ST_Point(0.5,1), 0.75) )
)
SELECT ST_IntersectionAgg(geom) FROM data;
```

#### Issues
* How to find all groups of intersecting polygons.  DBSCAN maybe?  (This is suggested in an answer)


### Intersection returning only Polygons
<https://gis.stackexchange.com/questions/89231/postgis-st-intersection-of-polygons-can-return-lines>

Possibly can use `ST_CollectionExtract`?

**See also**
* <https://gis.stackexchange.com/questions/242741/st-intersection-returns-erroneous-polygons> discusses a problem with QGIS visualization caused by the return of a `GEOMETRYCOLLECTION` from `ST_Intersection`.

### Intersection performance - Check containment first
https://postgis.net/2014/03/14/tip_intersection_faster/

### Intersection performance - Use Polygons instead of MultiPolygons
<https://gis.stackexchange.com/questions/101425/using-multipolygon-or-polygon-features-for-large-intersect-operations>

## Clipping

### Clip Districts to Coastline
http://blog.cleverelephant.ca/2019/07/simple-sql-gis.html

### Clip Unioned Polygons to remove water bodies
https://gis.stackexchange.com/questions/331887/unioning-a-set-of-intersections

