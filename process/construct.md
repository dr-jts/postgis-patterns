---
parent: Processing
---

# Constructions
{: .no_toc }

1. TOC
{:toc}

## Geodetic Constructions

### Construct ellipses in WGS84
<https://gis.stackexchange.com/questions/218159/postgis-ellipse-issue>

<https://gis.stackexchange.com/questions/409129/making-ellipses-on-a-global-data-set>

## Extending / Filling Polygons

### Construct polygons filling gaps in a coverage
<https://gis.stackexchange.com/questions/368406/postgis-create-new-polygons-in-between-existing>

![](https://i.stack.imgur.com/2LbzB.png)

```sql
SELECT ST_DIFFERENCE(foo.geom, bar.geom)
FROM (SELECT ST_CONVEXHULL(ST_COLLECT(shape::geometry)) as geom FROM schema.polytable) as foo, 
(SELECT ST_BUFFER(ST_UNION(shape),0.5) as geom FROM schema.polytable) as bar
```
To scale this up/out, could process batches of polygons using a rectangular grid defined over the data space. The constructed gap polygons can be clipped to grid cells. and optional unioned afterwards

### Remove Gaps between Polygons
<https://gis.stackexchange.com/questions/356480/enclose-polygons-that-are-not-overlapped-and-remove-gaps-between-them>
![](https://i.stack.imgur.com/XR8lU.png)

**Suggestions**

* Buffer positively and negatively (see `ST_BufferedUnion`in [PostGIS Addons](https://github.com/pedrogit/postgisaddons/blob/master/postgis_addons.sql) )
* [Answer](https://gis.stackexchange.com/a/316124/14766) using `ST_Buffer(geom, dist, 'join=mitre mitre_limit=5.0')` with pos/neg distance

### Construct polygon joining two polygons
<https://gis.stackexchange.com/questions/352884/how-can-i-get-a-polygon-of-everything-between-two-polygons-in-postgis>

![](https://i.stack.imgur.com/a7idE.png)

#### Solution
* construct convex hull of both polygons together
* subtract convex hull of each polygon
* union with original polygons
* keep only the polygon shell (remove holes)

```sql
WITH data(geom) AS (VALUES
( 'POLYGON ((100 300, 200 300, 200 200, 100 200, 100 300))'::geometry )
,( 'POLYGON ((50 150, 100 150, 100 100, 50 100, 50 150))'::geometry )
)
SELECT ST_MakePolygon(ST_ExteriorRing(ST_Union(
  ST_Difference( 
      ST_ConvexHull( ST_Union(geom)), 
      ST_Union( ST_ConvexHull(geom))), 
  ST_Collect(geom))))
FROM data;
```

### Construct expanded polygons to touches a bounding polygon
<https://gis.stackexchange.com/questions/294163/sql-postgis-expanding-polygons-contained-inside-another-polygon-until-one-ver>

![Expanding polygons to touch](https://i.stack.imgur.com/VQgHj.png)

```sql
WITH 
    to_buffer(distance, b.id) AS (
SELECT
   ST_Distance(ST_Exteriorring(a.geom), b.geom),
   b.id
 FROM 
    polygons_outer a, polygons_inner b 
 WHERE ST_Contains(a.geom, b.geom)
 UPDATE polygons_inner b
   SET geom = ST_Buffer(geom, distance)
  FROM to_buffer tb
  WHERE b.id = tb.id;
```
See also note about using a scaling rather than buffer, to preserve shape of polygon

## Grids

### Construct Land-Constrained Point Grid
<https://korban.net/posts/postgres/2019-10-17-generating-land-constrained-point-grids/>

![](https://korban.net/img/2019-10-16-12-44-48.png)

### Construct Square Grid
<https://gis.stackexchange.com/questions/16374/creating-regular-polygon-grid-in-postgis>

![](https://i.stack.imgur.com/vQY0Z.png)

<https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis>

<https://gis.stackexchange.com/questions/271234/creating-a-grid-on-a-polygon-and-find-number-of-points-in-each-grid>

## Medial Axis / Skeleton

### Construct Average / Centrelines between Lines
<https://gis.stackexchange.com/questions/322392/average-of-two-lines>

![](https://i.stack.imgur.com/QrEL8.png)

<https://gis.stackexchange.com/questions/50668/how-can-i-merge-collapse-nearby-and-parallel-road-lines-eg-a-dual-carriageway>

<https://github.com/migurski/Skeletron>

Idea: triangulate polygon, then connect midpoints of interior lines

Idea 2: find line segments for nearest points of each line vertex.  Order by distance along line (percentage?).  Discard any that have a retrograde direction.  Join centrepoints of segments.


### Construct Straight Skeleton
<https://github.com/twak/campskeleton>

## Polygon Diagonal / Radius

### Construct lines joining every vertex in a polygon

<https://gis.stackexchange.com/questions/58534/get-the-lines-between-all-points-of-a-polygon-in-postgis-avoid-nested-loop>

Better answer:
```sql
WITH poly(id, geom)AS (VALUES
    ( 1, 'POLYGON ((2 7, 5 9, 9 7, 8 3, 5 2, 2 3, 3 5, 2 7))'::geometry )
  )
,ring AS (SELECT ST_ExteriorRing(geom) geom FROM poly)
,pts AS (SELECT i, ST_PointN(geom, i) AS geom
  FROM ring
  JOIN LATERAL generate_series(2, ST_NumPoints(ring.geom)) AS s(i) ON true)
SELECT ST_MakeLine(p1.geom, p2.geom) AS geom
FROM pts p1, pts p2
WHERE p1.i > p2.i;
```

### Construct longest horizontal line within polygon
<https://gis.stackexchange.com/questions/32552/calculating-maximum-distance-within-polygon-in-x-direction-east-west-direction>

#### Algorithm
* For every Y value:
* Construct horizontal line across bounding box at Y value
* Intersect with polygon
* Keep longest result line

### Construct closest boundary point to a point in a polygon
<https://gis.stackexchange.com/questions/159318/finding-closest-outside-point-to-point-inside-polygon-in-postgis>

(Code below shows use of `ST_Buffer` to force the constructed point to lie outside the polygon.)

```sql
 WITH polygons AS(
    SELECT ST_Buffer(ST_GeomFromText('POINT(0 0)', 4326),2) as geom. -- example polygon
 )
 SELECT 
 ST_AsTEXT(ST_ClosestPoint(ST_ExteriorRing(ST_Buffer(polygons.geom,0.0000001))
        ,ST_GeomFromText('POINT(1 0)', 4326)))
FROM polygons;
```



