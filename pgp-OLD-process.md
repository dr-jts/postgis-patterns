---
nav_exclude: true
---

# Processing
{: .no_toc }

1. TOC
{:toc}



## Constructions

### Construct polygons filling gaps in a coverage
https://gis.stackexchange.com/questions/368406/postgis-create-new-polygons-in-between-existing

```sql
SELECT ST_DIFFERENCE(foo.geom, bar.geom)
FROM (SELECT ST_CONVEXHULL(ST_COLLECT(shape::geometry)) as geom FROM schema.polytable) as foo, 
(SELECT ST_BUFFER(ST_UNION(shape),0.5) as geom FROM schema.polytable) as bar
```
To scale this up/out, could process batches of polygons using a rectangular grid defined over the data space. The constructed gap polygons can be clipped to grid cells. and optional unioned afterwards

### Construct ellipses in WGS84
https://gis.stackexchange.com/questions/218159/postgis-ellipse-issue

### Construct polygon joining two polygons
https://gis.stackexchange.com/questions/352884/how-can-i-get-a-polygon-of-everything-between-two-polygons-in-postgis

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
https://gis.stackexchange.com/questions/294163/sql-postgis-expanding-polygons-contained-inside-another-polygon-until-one-ver

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

### Construct Land-Constrained Point Grid
https://korban.net/posts/postgres/2019-10-17-generating-land-constrained-point-grids/

### Construct Square Grid
https://gis.stackexchange.com/questions/16374/creating-regular-polygon-grid-in-postgis

https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis

https://gis.stackexchange.com/questions/271234/creating-a-grid-on-a-polygon-and-find-number-of-points-in-each-grid

### Construct Polygon Centrelines
https://gis.stackexchange.com/questions/322392/average-of-two-lines?noredirect=1&lq=1

https://gis.stackexchange.com/questions/50668/how-can-i-merge-collapse-nearby-and-parallel-road-lines-eg-a-dual-carriageway

https://github.com/migurski/Skeletron


Idea: triangulate polygon, then connect midpoints of interior lines

Idea 2: find line segments for nearest points of each line vertex.  Order by distance along line (percentage?).  Discard any that have a retrograde direction.  Join centrepoints of segments.

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

### Construct Straight Skeleton
<https://github.com/twak/campskeleton>

## Hulls / Covering Polygons

### Construct Bounding box of set of MULTILINESTRINGs
https://gis.stackexchange.com/questions/115494/bounding-box-of-set-of-multilinestrings-in-postgis
```sql
SELECT type, ST_Envelope(ST_Collect(geom))
FROM line_table AS foo
GROUP BY type;
```

### Construct polygon containing lines
https://gis.stackexchange.com/questions/238/find-polygon-that-contains-all-linestring-records-in-postgis-table

### Construct lines between all points of a Polygon
https://gis.stackexchange.com/questions/58534/get-the-lines-between-all-points-of-a-polygon-in-postgis-avoid-nested-loop

##### Solution
Rework given SQL using CROSS JOIN and a self join

### Construct Regions from Points
https://gis.stackexchange.com/questions/92913/extra-detailed-bounding-polygon-from-many-geometric-points

### Construct regions from large sets of points (100K) tagged with region attribute.

Could use ST_ConcaveHull, but results would overlap
Perhaps ST_Voronoi would be better?  How would this work, and what are limits on size of data?

### Construct a Star Polygon from a set of Points
https://gis.stackexchange.com/questions/349945/creating-precise-shapes-using-list-of-coordinates

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

## Buffers

### Variable Width Buffer
https://gis.stackexchange.com/questions/340968/varying-size-buffer-along-a-line-with-postgis

### Expand a rectangular polygon
https://gis.stackexchange.com/questions/308333/expanding-polygon-by-distance-using-postgis

### Buffer Coastlines with inlet skeletons
https://gis.stackexchange.com/questions/300867/how-can-i-buffer-a-mulipolygon-only-on-the-coastline

### Remove Line Buffer artifacts
https://gis.stackexchange.com/questions/363025/how-to-run-a-moving-window-function-in-a-conditional-statement-in-postgis-for-bu

Quite bizarre, but apparently works.


## Generating Point Distributions
### Generate Evenly-Distributed Points in a Polygon

<https://gis.stackexchange.com/questions/8468/creating-evenly-distributed-points-within-an-irregular-boundary>

One solution: create a grid of points and then clip to polygon

See also: 
* <https://math.stackexchange.com/questions/15624/distribute-a-fixed-number-of-points-uniformly-inside-a-polygon>
* <https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis>

### Construct Well-spaced Random Points in a Polygon
<https://gis.stackexchange.com/questions/377606/ensuring-all-points-are-a-certain-distance-from-polygon-boundary>

Uses clustering on randomly generated points.  
Suggestion is to use neg-buffered polygon to ensure distance from polygon boundary

A nicer solution will be when PostGIS provides a way to generate random points using [Poisson Disk Sampling](https://www.jasondavies.com/poisson-disc/).

### Place Maximum Number of Points in a Polygon
<https://gis.stackexchange.com/questions/4828/seeking-algorithm-to-place-maximum-number-of-points-within-constrained-area-at-m>

### Sample Linear Point Clouds at given density
<https://gis.stackexchange.com/questions/131854/spacing-a-set-of-points>

## Contouring
### Generate contours from evenly-spaced weighted points
https://gis.stackexchange.com/questions/85968/clustering-points-in-postgresql-to-create-contour-map
NO SOLUTION

### Contouring Irregularly spaced points
https://abelvm.github.io/sql/contour/

Solution
An impressive PostGIS-only solution using a Delaunay with triangles cut by contour lines.
Uses the so-called Meandering Triangles method for isolines.

## Surface Interpolation

### IDW Interpolation over a grid of points
https://gis.stackexchange.com/questions/373153/spatial-interpolation-in-postgis-without-outputting-raster


## Conflation / Matching

### Adjust polygons to fill a containing Polygon
<https://gis.stackexchange.com/questions/91889/adjusting-polygons-to-boundary-and-filling-holes>

### Match Polygons by Shape Similarity 
https://gis.stackexchange.com/questions/362560/measuring-the-similarity-of-two-polygons-in-postgis

There are different ways to measure the similarity between two polygons such as average distance between the boundaries, Hausdorff distance, Turning Function, Comparing Fourier Transformation of the two polygons

Gives code for Average Boundary Distance

### Find Polygon with more accurate linework
https://gis.stackexchange.com/questions/257052/given-two-polygons-find-the-the-one-with-more-detailed-accurate-shoreline

### Match sets of LineStrings
https://gis.stackexchange.com/questions/347787/compare-two-set-of-linestrings

### Match paths to road network
https://gis.stackexchange.com/questions/349001/aligning-line-with-closest-line-segment-in-postgis

### Match paths
https://gis.stackexchange.com/questions/368146/matching-segments-within-tracks-in-postgis

### Polygon Averaging
https://info.crunchydata.com/blog/polygon-averaging-in-postgis

Solution: Overlay, count “depth” of each resultant, union resultants of desired depth.





