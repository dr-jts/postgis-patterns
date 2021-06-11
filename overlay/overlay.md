---
parent: Overlay
---

# Overlay
{: .no_toc }

1. TOC
{:toc}

## Overlay - Overlapping Polygon Dataset

### Create coverage from Nested Polygons
<https://gis.stackexchange.com/questions/266005/postgis-separate-nested-polygons>

### Create Coverage from Overlapping Polygons
<https://gis.stackexchange.com/questions/83/separate-polygons-based-on-intersection-using-postgis>
<https://gis.stackexchange.com/questions/112498/postgis-overlay-style-union-not-dissolve-style>
<https://gis.stackexchange.com/questions/109692/how-to-replicate-arcgis-intersect-in-postgis>

#### Solution
One answer suggests the standard Extract Lines > Node > Polygonize approach (although does not include the PIP parentage step).  But a comment says that this does not scale well (Pierre Racine…).
Also links to PostGIS wiki:  <https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables>


### Count Overlap Depth in set of polygons

<http://blog.cleverelephant.ca/2019/07/postgis-overlays.html>

![](http://blog.cleverelephant.ca/images//2019/overlays8.png)

Howver, this post indicates this approach might be slow for large datasets:
<https://gis.stackexchange.com/questions/159282/counting-overlapping-polygons-in-postgis-using-st-union-very-slow>

#### Solution
* Extract linework of polygons using `ST_Boundary`
* Compute overlay of dataset using `ST_Union` 
  * sometimes `ST_Node` is suggested, but this does not dissolve linework, which causes problems with polgonization 
* Polygonize linwork using `ST_Polygonize`
* Generate an interior point for each resultant using `ST_PointOnSurface`
* Count resultant overlap depth by joining back to original dataset with `ST_Contains`

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

## Overlay - Two Polygonal Coverages

### Overlay of two polygon layers - by Intersection

Solution retains all areas from first layer, but not the second:
<https://gis.stackexchange.com/questions/179533/arcgis-union-equivalent-in-postgis>

This has what looks like a complete solution (although complex):
<https://gis.stackexchange.com/questions/401245/getting-the-composite-of-two-polygon-layers<

#### See also
This answer seems suspect - it may be doing more work than required.

<https://gis.stackexchange.com/questions/302086/postgis-union-of-two-polygons-layers>

### Overlay of two polygon layers - by Node-Polygonize

* Extract linework using `ST_Boundary`
* Node/dissolve lines using `ST_Union`
  * Sometimes `ST_Node` is suggested, but it does not dissolve duplicate lines 
* Polygonize resultants using `ST_Polygonize`
* Generate an interior point for each resultant using `ST_PointOnSurface`
* Attach parent attribution by joining on interior points using `ST_Contains`

#### Notes
* All intermediate operations are dataset-wide, so materializing as intermediate tables will not improve performance.  That might help with memory usage, however.
* The input tables should have spatial indexes, to improve performance of the final join step
* This approach generalizes nicely to multiple tables,

<https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesOverlayTables>

```sql
WITH poly_a(id, geom) AS (VALUES
    ( 'a1', 'POLYGON ((10 40, 30 40, 30 10, 10 10, 10 40))'::geometry ),
    ( 'a2', 'POLYGON ((70 10, 30 10, 30 90, 70 90, 70 10), (40 40, 60 40, 60 20, 40 20, 40 40), (40 80, 60 80, 60 60, 40 60, 40 80))'::geometry ),
    ( 'a3', 'POLYGON ((40 40, 60 40, 60 20, 40 20, 40 40))'::geometry )
)
,poly_b(id, geom) AS (VALUES
    ( 'b1', 'POLYGON ((90 70, 90 50, 50 50, 50 70, 90 70))'::geometry ),
    ( 'b2', 'POLYGON ((90 30, 50 30, 50 50, 90 50, 90 30))'::geometry ),
    ( 'b2', 'POLYGON ((90 10, 70 10, 70 30, 90 30, 90 10))'::geometry )
)
,lines AS ( 
  SELECT ST_Boundary(geom) AS geom FROM poly_a
  UNION ALL
  SELECT ST_Boundary(geom) AS geom FROM poly_b
)
,noded_lines AS ( SELECT ST_Union(geom) AS geom FROM lines ) 
,resultants AS (  
  SELECT geom, ST_PointOnSurface(geom) AS pip 
    FROM St_Dump(
           ( SELECT ST_Polygonize(geom) AS geom FROM noded_lines ))   
)
SELECT a.id AS ida, b.id AS idb, r.geom
  FROM resultants r
  LEFT JOIN poly_a a ON ST_Contains(a.geom, r.pip) 
  LEFT JOIN poly_b b ON ST_Contains(b.geom, r.pip)
  WHERE a.id IS NOT NULL OR b.id IS NOT NULL;
```
![image](https://user-images.githubusercontent.com/3529053/121740578-1fb6df80-cab2-11eb-93a5-eb28966766cf.png)
![image](https://user-images.githubusercontent.com/3529053/121740709-5b51a980-cab2-11eb-8b78-d74e30aede65.png)


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

### Sum rectangular grid values weighted by area of overlap with Polygons
<https://gis.stackexchange.com/questions/171333/weighting-amount-of-overlapping-polygons-in-postgis>

![](https://i.stack.imgur.com/NQ8zu.jpg)

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



