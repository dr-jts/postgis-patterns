# Processing
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


## Geometry Editing

### Drop Holes from Polygons
https://gis.stackexchange.com/questions/278154/polygons-have-holes-after-pgr-pointsaspolygon

### Drop Holes from MultiPolygons
https://gis.stackexchange.com/questions/348943/simplifying-a-multipolygon-into-one-polygon-respecting-its-outer-boundaries

Solution
https://gis.stackexchange.com/a/349016/14766

Similar
https://gis.stackexchange.com/questions/291374/cut-out-polygons-that-at-least-partially-fall-in-another-polygon


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


## Measuring

### Find Median width of Road Polygons
https://gis.stackexchange.com/questions/364173/calculate-median-width-of-road-polygons

### Unbuffering - find average distance from a buffer and source polygon
https://gis.stackexchange.com/questions/33312/is-there-a-st-buffer-inverse-function-that-returns-a-width-estimation

Also: https://gis.stackexchange.com/questions/20279/calculating-average-width-of-polygon

### Compute Length and Width of an arbitrary rectangle
https://gis.stackexchange.com/questions/366832/get-dimension-of-rectangular-polygon-postgis


## Simplification
https://gis.stackexchange.com/questions/293429/decrease-polygon-vertices-count-maintaining-its-aspect

## Smoothing
https://gis.stackexchange.com/questions/313667/postgis-snap-line-segment-endpoint-to-closest-other-line-segment

Problem is to smooth a network of lines.  Network is not fully noded, so smoothing causes touching lines to become disconnected.
#### Solution
Probably to node the network before smoothing.
Not sure how to node the network and preserve IDs however!?

## Transformation

### Scale polygon around a given point
https://gis.stackexchange.com/questions/227435/postgis-scaling-for-polygons-at-a-fixed-center-location

No solution so far
Issues
SQL given is overly complex and inefficient.  But idea is right

## Ordering Geometry

### Ordering a Square Grid
https://gis.stackexchange.com/questions/346519/sorting-polygons-into-a-n-x-n-spatial-array

https://gis.stackexchange.com/questions/255512/automatically-name-rectangles-by-spatial-order-or-position-to-each-other?noredirect=1&lq=1

### Serpentine Ordering
https://gis.stackexchange.com/questions/176197/seeking-tool-algorithm-for-assigning-code-to-enumeration-areas-polygons-using

No solution in the post

Also
https://gis.stackexchange.com/questions/73978/numbering-polygons-according-to-their-spatial-relationships

### Ordering Polygons along a line
https://gis.stackexchange.com/questions/201306/numbering-adjacent-polygons-in-sequential-order

No explicit solution given, but suggestion is to compute adjacency graph and then do a graph traversal

### Order polygons touched by a Line
https://gis.stackexchange.com/questions/317401/maintaining-order-and-repetition-of-cell-names-using-postgis

### Connecting Circles Into a Polygonal Path
https://gis.stackexchange.com/questions/246521/connecting-circles-with-lines-cover-all-circles-using-postgis

### Ordered list of polygons intersecting a line
https://gis.stackexchange.com/questions/179061/find-all-intersections-of-a-linestring-and-a-polygon-and-the-order-in-which-it-i

### Ordering points along lines

Given:
* way (id, name, geom (multistring))
* station (id, name, geom (polygon)) - circles for each station, which may or may not intersect way line

Find the station order along each way

https://gis.stackexchange.com/questions/379220/finding-station-order-number-along-way-using-postgis

The first step is to replace the station polygons by points (i.e. use centroid)

```sql
WITH
  dumped_ways AS (
    SELECT way_name,
           dmp.path[1] AS way_part,
           dmp.geom
    FROM   s_602_ptrc.way,
           LATERAL ST_Dump(geom) AS dmp
  )

SELECT ROW_NUMBER() OVER(PARTITION BY nway.way_name, nway.way_part ORDER BY nway._frac) AS id,
       station.nome AS station,
       nway.way_name,
       nway.way_part,
       station.geom
FROM   s_602_ptrc.station
CROSS JOIN LATERAL (
  SELECT dumped_ways.way_name,
         dumped_ways.way_part,
         ST_LineLocatePoint(dumped_ways.geom, station.geom) AS _frac
  FROM   dumped_ways
  ORDER BY
         dumped_ways.geom <-> ST_Centroid(station.geom)
  LIMIT  1
) AS nway
ORDER BY nway.way_name, nway.way_part, nway._frac;
```

## Generating Point Distributions
### Generate Evenly-Distributed Points in a Polygon

https://gis.stackexchange.com/questions/8468/creating-evenly-distributed-points-within-an-irregular-boundary

One solution: create a grid of points and then clip to polygon

See also: https://math.stackexchange.com/questions/15624/distribute-a-fixed-number-of-points-uniformly-inside-a-polygon
https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis

### Construct Well-spaced Random Points in a Polygon
<https://gis.stackexchange.com/questions/377606/ensuring-all-points-are-a-certain-distance-from-polygon-boundary>

Uses clustering on randomly generated points.  
Suggestion is to use neg-buffered polygon to ensure distance from polygon boundary

A nicer solution will be when PostGIS provides a way to generate random points using [Poisson Disk Sampling](https://www.jasondavies.com/poisson-disc/).

### Place Maximum Number of Points in a Polygon
https://gis.stackexchange.com/questions/4828/seeking-algorithm-to-place-maximum-number-of-points-within-constrained-area-at-m

### Thin out Points along lines
https://gis.stackexchange.com/questions/131854/spacing-a-set-of-points

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


## Coordinate Systems

### Find a good planar projection
https://gis.stackexchange.com/questions/275057/how-to-find-good-meter-based-projection-in-postgis

Also https://gis.stackexchange.com/questions/341243/postgis-buffer-in-meters-without-geography

### ST_Transform creates invalid geometry
https://gis.stackexchange.com/questions/341160/why-do-two-tables-with-valid-geometry-return-self-intersection-errors-when-inter

Also:  https://trac.osgeo.org/postgis/ticket/4755
Has an example geometry which becomes invalid under transform



