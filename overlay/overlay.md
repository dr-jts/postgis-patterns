---
parent: Overlay
---

# Overlay
{: .no_toc }

1. TOC
{:toc}

## Overlay - Single Input

### Flatten / Create coverage from Nested Polygons
<https://gis.stackexchange.com/questions/266005/postgis-separate-nested-polygons>

### Create Coverage from overlapping Polygons
<https://gis.stackexchange.com/questions/83/separate-polygons-based-on-intersection-using-postgis>
<https://gis.stackexchange.com/questions/112498/postgis-overlay-style-union-not-dissolve-style>
<https://gis.stackexchange.com/questions/109692/how-to-replicate-arcgis-intersect-in-postgis>
<http://blog.cleverelephant.ca/2019/07/postgis-overlays.html>

#### Solution
One answer suggests the standard Extract Lines > Node > Polygonize approach (although does not include the PIP parentage step).  But a comment says that this does not scale well (Pierre Racine…).
Also links to PostGIS wiki:  <https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables>


### Count Overlap Depth in set of polygons
<https://gis.stackexchange.com/questions/159282/counting-overlapping-polygons-in-postgis-using-st-union-very-slow>

#### Solution
* Compute overlay of dataset using `ST_Node` and `ST_Polygonize`.
* Count overlap depth using `ST_PointOnSurface` and `ST_Contains`

### Identify Overlapping Polygon Resultant Parentage

<https://gis.stackexchange.com/questions/315368/listing-all-overlapping-polygons-using-postgis>



### Return only polygons from Overlay
<https://gis.stackexchange.com/questions/89231/postgis-st-intersection-of-polygons-can-return-lines>

<https://gis.stackexchange.com/questions/242741/st-intersection-returns-erroneous-polygons>

### Compute Coverage from Overlapping Polygons
<https://gis.stackexchange.com/questions/206473/obtaining-each-unique-area-of-overlapping-polygons-in-postgres-9-6-postgis-2-3>

#### Problem
Reduce a dataset of highly overlapping polygons to a coverage (not clear if attribution is needed or not)

#### Issues
User implements a very complex overlay process, but can not get it to work, likely due to robustness problems

#### Solution
`ST_Boundary` -> `ST_Union` -> `ST_Polygonize` ??

## Overlay - Two Inputs

### Overlay of two polygon layers - by Intersection

<https://gis.stackexchange.com/questions/179533/arcgis-union-equivalent-in-postgis>

Wants a coverage overlay (called “QGIS Union”)
<https://gis.stackexchange.com/questions/302086/postgis-union-of-two-polygons-layers>

See also

<https://gis.stackexchange.com/questions/115927/is-there-a-union-function-for-multiple-layers-comparable-to-arcgis-in-open-sourc>

### Overlay of two polygon layers - by Node-Polygonize

<https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables>

![](https://trac.osgeo.org/postgis/raw-attachment/wiki/UsersWikiExamplesOverlayTables/overlay1.png)

### Remove Polygons overlapped area on update to Polygon in other table 
<https://gis.stackexchange.com/questions/90174/postgis-when-i-add-a-polygon-delete-overlapping-areas-in-other-layers>

### Improve performance of a coverage overlay
<https://gis.stackexchange.com/questions/31310/acquiring-arcgis-like-speed-in-postgis>

#### Problem
Finding all intersections of a large set of parcel polygons against a set of jurisdiction polygons is slow

#### Solution
Reduce # calls to ST_Intersection by testing if parcel is wholly contained in polygon. 
```sql
INSERT INTO parcel_jurisdictions(parcel_gid,jurisdiction_gid,isect_geom) 
  SELECT a.orig_gid AS parcel_gid, b.orig_gid AS jurisdiction_gid, 
    CASE 
      WHEN ST_Within(a.geom,b.geom) THEN a.geom 
      ELSE ST_Multi(ST_Intersection(a.geom,b.geom)) 
    END AS geom 
  FROM valid_parcels a 
    JOIN valid_jurisdictions b ON ST_Intersects(a.geom, b.geom);
```
References
<https://postgis.net/2014/03/14/tip_intersection_faster/>

### Sum of polygonal grid values weighted by area of intersection with polygon
<https://gis.stackexchange.com/questions/171333/weighting-amount-of-overlapping-polygons-in-postgis>

### Sum Percentage of Polygons intersected by other Polygons, by label

Given two polygonal coverages A and B with labels, compute the percentage area of each polygon in A covered by each label in B.

<https://gis.stackexchange.com/questions/378532/working-out-percentage-of-polygon-covering-another-using-postgis>


![](https://i.stack.imgur.com/bQjSi.png)

```sql
WITH poly_a(name, geom) AS (VALUES
    ( 'a1', 'POLYGON ((100 200, 200 200, 200 100, 100 100, 100 200))'::geometry ),
    ( 'a2', 'POLYGON ((300 300, 300 200, 200 200, 200 300, 300 300))'::geometry ),
    ( 'a3', 'POLYGON ((400 400, 400 300, 300 300, 300 400, 400 400))'::geometry )
),
poly_b(name, geom) AS (VALUES
    ( 'b1', 'POLYGON ((120 280, 280 280, 280 120, 120 120, 120 280))'::geometry ),
    ( 'b2', 'POLYGON ((280 280, 280 320, 320 320, 320 280, 280 280))'::geometry ),
    ( 'b2', 'POLYGON ((390 390, 390 360, 360 360, 360 390, 390 390))'::geometry )
)
SELECT a.name, b.name, SUM( ST_Area(ST_Intersection(a.geom, b.geom))/ST_Area(a.geom) ) pct
FROM poly_a a JOIN poly_b b ON ST_Intersects(a.geom, b.geom)
GROUP BY a.name, b.name;
```



