# PostGIS Patterns - Miscellaneous

* Geometry Creation
* Geometry Editing
* Constructions
* Surface Interpolation

## Geometry Creation

### Use ST_MakePoint or ST_PointFromText
[](https://gis.stackexchange.com/questions/122247/st-makepoint-or-st-pointfromtext-to-generate-points)
https://gis.stackexchange.com/questions/58605/which-function-for-creating-a-point-in-postgis/58630#58630

Solutions
ST_MakePoint is much faster

### Collect Lines into a MultiLine in a given order
https://gis.stackexchange.com/questions/166701/postgis-merging-linestrings-into-multilinestrings-in-a-particular-order



## Geometry Editing

### Drop Holes from Polygons
https://gis.stackexchange.com/questions/278154/polygons-have-holes-after-pgr-pointsaspolygon

### Drop Holes from MultiPolygons
https://gis.stackexchange.com/questions/348943/simplifying-a-multipolygon-into-one-polygon-respecting-its-outer-boundaries

Solution
https://gis.stackexchange.com/a/349016/14766

Similar
https://gis.stackexchange.com/questions/291374/cut-out-polygons-that-at-least-partially-fall-in-another-polygon

### Select every Nth point from a LineString
https://stackoverflow.com/questions/60319473/postgis-how-do-i-select-every-second-point-from-linestring



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

### Construct Straight Skeleton
https://github.com/twak/campskeleton

### Construct Well-spaced points within Polygon
https://gis.stackexchange.com/questions/377606/ensuring-all-points-are-a-certain-distance-from-polygon-boundary

Uses clustering on randomly generated points.  
Suggestion is to use neg-buffered polygon to ensure distance from polygon boundary

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
https://gis.stackexchange.com/questions/92913/extra-detailed-bounding-polygon-from-many-geometric-points?rq=1

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
https://gis.stackexchange.com/questions/179061/find-all-intersections-of-a-linestring-and-a-polygon-and-the-order-in-which-it-i?rq=1


## Generating Point Distributions
### Generate Evenly-Distributed Points in a Polygon

https://gis.stackexchange.com/questions/8468/creating-evenly-distributed-points-within-an-irregular-boundary?rq=1

One solution: create a grid of points and then clip to polygon

See also: https://math.stackexchange.com/questions/15624/distribute-a-fixed-number-of-points-uniformly-inside-a-polygon
https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis

### Place Maximum Number of Points in a Polygon
https://gis.stackexchange.com/questions/4828/seeking-algorithm-to-place-maximum-number-of-points-within-constrained-area-at-m

### Thin out Points along lines
https://gis.stackexchange.com/questions/131854/spacing-a-set-of-points?rq=1

## Contouring
### Generate contours from evenly-spaced weighted points
https://gis.stackexchange.com/questions/85968/clustering-points-in-postgresql-to-create-contour-map?rq=1
NO SOLUTION

### Contouring Irregularly spaced points
https://abelvm.github.io/sql/contour/

Solution
An impressive PostGIS-only solution using a Delaunay with triangles cut by contour lines.
Uses the so-called Meandering Triangles method for isolines.

## Clustering
See https://gis.stackexchange.com/questions/11567/spatial-clustering-with-postgis for a variety of approaches that predate a lot of the PostGIS clustering functions.

### Grid-Based Clustering
https://gis.stackexchange.com/questions/352562/is-it-possible-to-get-one-geometry-per-area-box-in-postgis

Solution 1
Use ST_SnapToGrid to compute a cell id for each point, then bin the points based on that.  Can use aggregate function to count points in grid cell, or use DISTINCT ON as a cheesy way to pick one representative point.  Need to use representative point rather than average, for better visual results (perhaps?)

Solution 2
Generate grid of cells covering desired area, then JOIN LATERAL to points to aggregate.  Not sure how to select a representative point doing this though - perhaps MIN or MAX?  Requires a grid-generating function, which is coming in PostGIS 3.1

### Non-spatial clustering by distance
https://stackoverflow.com/questions/49250734/sql-window-function-that-groups-values-within-selected-distance

### Find Density Centroids Within Polygons
https://gis.stackexchange.com/questions/187256/finding-density-centroids-within-polygons-in-postgis?rq=1

### Group touching Polygons
https://gis.stackexchange.com/questions/343514/postgis-recursively-finding-intersections-of-polygons-to-determine-clusters
Solution: Use ST_DBSCAN which provides very good performance

This problem has a similar recommended solution:
https://gis.stackexchange.com/questions/265137/postgis-union-geometries-that-intersect-but-keep-their-original-geometries-info

A worked example:
https://gis.stackexchange.com/questions/366374/how-to-use-dissolve-a-subset-of-a-postgis-table-based-on-a-value-in-a-column

Similar problem in R
https://gis.stackexchange.com/questions/254519/group-and-union-polygons-that-share-a-border-in-r?rq=1

Issues
DBSCAN uses distance. This will also cluster polygons which touch only at a point, not just along an edge.  Is there a way to improve this?  Allow a different distance metric perhaps - say length of overlap?

### Group connected LineStrings
https://gis.stackexchange.com/questions/94203/grouping-connected-linestrings-in-postgis

Presents a recursive CTE approach, but ultimately recommends using ST_ClusterDBCSAN

https://gis.stackexchange.com/questions/189091/postgis-how-to-merge-contiguous-features-sharing-same-attributes-values

### Kernel Density
https://gist.github.com/AbelVM/dc86f01fbda7ba24b5091a7f9b48d2ee

### Group Polygon Coverage into similar-sized Areas
https://gis.stackexchange.com/questions/350339/how-to-create-polygons-of-a-specific-size

More generally: how to group adjacent polygons into sets with similar sum of a given attribute.

See Also
https://gis.stackexchange.com/questions/123289/grouping-village-points-based-on-distance-and-population-size-using-arcgis-deskt

Solution
Build adjacency graph and aggregate based on total, and perhaps some distance criteria?
Note: posts do not provide a PostGIS solution for this.  Not known if such a solution exists.
Would need a recursive query to do this.
How to keep clusters compact?

### Bottom-Up Clustering Algorithm
Not sure if this is worthwhile or not.  Possibly superseded by more recent standard PostGIS clustering functions

https://gis.stackexchange.com/questions/113545/get-a-single-cluster-from-cloud-of-points-with-specified-maximum-diameter-in-pos

### Using ClusterWithin VS ClusterDBSCAN
https://gis.stackexchange.com/questions/348484/clustering-points-in-postgis

Explains how DBSCAN is a superset of ClusterWithin, and provides simpler, more powerful SQL.

### Removing Clusters of Points
https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius

### Cluster with DBSCAN partitioned by polygons
https://gis.stackexchange.com/questions/284190/python-cluster-points-with-dbscan-keeping-into-account-polygon-boundaries?rq=1

### Compute centroid of a group of points
https://gis.stackexchange.com/questions/269407/centroid-of-point-cluster-points

### FInd polygons which are not close to any other polygon
https://gis.stackexchange.com/questions/312167/calculating-shortest-distance-between-polygons
Solution
Use `ST_GeometricMedian`

### Cluster with DBSCAN partitioned by record types
https://gis.stackexchange.com/questions/357838/how-to-cluster-points-with-st-clusterdbscan-taking-into-account-their-type-store

### Select evenly-distributed points of unevenly-distributed set, with priority
https://gis.stackexchange.com/questions/346412/select-evenly-distirbuted-points-of-unevenly-distributed-set

### Construct K-Means clusters for each Polygon
https://gis.stackexchange.com/questions/376563/cluster-points-in-each-polygon-into-n-parts 

Use window function PARTITION BY



## Surface Interpolation'

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
https://gis.stackexchange.com/questions/91889/adjusting-polygons-to-boundary-and-filling-holes?rq=1

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

## Routing / Pathfinding

### Seaway Distances & Routes
https://www.ausvet.com.au/seaway-distances-with-postgresql/
![](https://www.ausvet.com.au/wp-content/uploads/Blog_images/seaway_1.png)

## Linear Referencing/Line Handling
### Extrapolate Lines
https://gis.stackexchange.com/questions/33055/extrapolating-a-line-in-postgis

ST_LineInterpolatePoint should be enhanced to allow fractions outside [0,1].

### Extend a LineString to the boundary of a polygon
https://gis.stackexchange.com/questions/345463/how-can-i-extend-a-linestring-to-the-edge-of-an-enclosing-polygon-in-postgis

Ideas
A function ST_LineExtract(line, index1, index2) to extract a portion of a LineString between two indices

### Compute Angle at which Two Lines Intersect
https://gis.stackexchange.com/questions/25126/how-to-calculate-the-angle-at-which-two-lines-intersect-in-postgis?rq=1

### Compute Azimuth at a Point on a Line
https://gis.stackexchange.com/questions/178687/rotate-point-along-line-layer

Solution
Would be nice to have a ST_SegmentIndex function to get the index of the segment nearest to a point.  Then this becomes simple.

### Compute Perpendicular Distance to a Baseline (AKA “Width” of a curve)
https://gis.stackexchange.com/questions/54575/how-to-calculate-the-depth-of-a-linestring-using-postgis?rq=1

### Measure Length of every LineString segment
https://gis.stackexchange.com/questions/239576/measure-length-of-each-segment-for-a-polygon-in-postgis
Solution
Use CROSS JOIN LATERAL with generate_series and ST_MakeLine, ST_Length
ST_DumpSegments would make this much easier!

### Find Lines that form Rings
https://gis.stackexchange.com/questions/32224/select-the-lines-that-form-a-ring-in-postgis
Solutions
Polygonize all lines, then identify lines which intersect each polygon
Complicated recursive solution using ST_LineMerge!

### Add intersection points between sets of Lines (AKA Noding)
https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis?rq=1
Problem
Add nodes into a road network from access side roads

Solutions
One post recommends simply unioning (overlaying) all the linework.  This was accepted, but has obvious problems:
Hard to extract just the road network lines
If side road falls slightly short will not create a node

### Merge lines that touch at endpoints
https://gis.stackexchange.com/questions/177177/finding-and-merging-lines-that-touch-in-postgis?rq=1

Solution given uses ST_ClusterWithin, which is clever.  Can be improved slightly however (e.g. can use ST_Boundary to get endpoints?).  Would be much nicer if ST_ClusterWithin was a window function.  

Could also use a recursive query to do a transitive closure of the “touches at endpoints” condition.  This would be a nice example, and would scale better.

Can also use ST_LineMerge to do this very simply (posted).

Also:

https://gis.stackexchange.com/questions/360795/merge-linestring-that-intersects-without-making-them-multilinestring
```sql
WITH data(geom) AS (VALUES
( 'LINESTRING (50 50, 150 100, 250 75)'::geometry )
,( 'LINESTRING (250 75, 200 0, 130 30, 100 150)'::geometry )
,( 'LINESTRING (100 150, 130 170, 220 190, 290 190)'::geometry )
)
SELECT ST_AsText(ST_LineMerge(ST_Collect(geom))) AS line 
FROM data;
```
### Merge lines that touch at endpoints 2
https://gis.stackexchange.com/questions/16698/join-intersecting-lines-with-postgis

Solution
https://gis.stackexchange.com/a/80105/14766

### Merge lines which do not form a single line
https://gis.stackexchange.com/questions/83069/cannot-st-linemerge-a-multilinestring-because-its-not-properly-ordered
Solution
Not possible with ST_LineMerge
Error is not obvious from return value though

See Also
https://gis.stackexchange.com/questions/139227/st-linemerge-doesnt-return-linestring?rq=1

### Extract Line Segments
https://gis.stackexchange.com/questions/174472/in-postgis-how-to-split-linestrings-into-their-individual-segments?rq=1

Need an ST_DumpSegments to do this!

### Remove Longest Segment from a LineString
https://gis.stackexchange.com/questions/372110/postgis-removing-the-longest-segment-of-a-linestring-and-rejoining-segments

Solution (part)
Remove longest segment, splitting linestring into two parts if needed.

Useful patterns in this code:

JOIN LATERAL generate_series to extract the line segments
array slicing to extract a subline containing a section of the original line

It would be clearer if parts of this SQL were wrapped in functions (e.g. perhaps an ST_LineSlice function, and a ST_DumpSegments function - which perhaps will become part of PostGIS one day).
```sql
WITH data(id, geom) AS (VALUES
    ( 1, 'LINESTRING (0 0, 1 1, 2.1 2, 3 3, 4 4)'::geometry )
),
longest AS (SELECT i AS iLongest, geom,
    ST_Distance(  ST_PointN( data.geom, s.i ),
                  ST_PointN( data.geom, s.i+1 ) ) AS dist
   FROM data JOIN LATERAL (
        SELECT i FROM generate_series(1, ST_NumPoints( data.geom )-1) AS gs(i)
     ) AS s(i) ON true
   ORDER BY dist LIMIT 1
)
SELECT 
  CASE WHEN iLongest > 2 THEN ST_AsText( ST_MakeLine(
    (ARRAY( SELECT (ST_DumpPoints(geom)).geom FROM longest))[1 : iLongest - 1]
  )) ELSE null END AS line1,
  CASE WHEN iLongest < ST_NumPoints(geom) - 1 THEN ST_AsText( ST_MakeLine(
    (ARRAY( SELECT (ST_DumpPoints(geom)).geom FROM longest))[iLongest + 1: ST_NumPoints(geom)]
  )) ELSE null END AS line2
FROM longest;
```
### Split Lines into Equal-length portions
https://gis.stackexchange.com/questions/97990/break-line-into-100m-segments/334305#334305

Modern solution using LATERAL:
```sql
WITH data AS (
    SELECT * FROM (VALUES
        ( 'A', 'LINESTRING( 0 0, 200 0)'::geometry ),
        ( 'B', 'LINESTRING( 0 100, 350 100)'::geometry ),
        ( 'C', 'LINESTRING( 0 200, 50 200)'::geometry )
    ) AS t(id, geom)
)
SELECT ST_LineSubstring( d.geom, substart, 
    CASE WHEN subend > 1 THEN 1 ELSE subend END ) geom
FROM (SELECT id, geom, ST_Length(geom) len, 100 sublen FROM data) AS d
CROSS JOIN LATERAL (
    SELECT i,  
            (sublen * i)/len AS substart,
            (sublen * (i+1)) / len AS subend
        FROM generate_series(0, 
            floor( d.len / sublen )::integer ) AS t(i)
        WHERE (sublen * i)/len <> 1.0  
    ) AS d2;
```
 Need to update PG doc:  https://postgis.net/docs/ST_LineSubstring.html

See also 
https://gis.stackexchange.com/questions/346196/split-a-linestring-by-distance-every-x-meters-using-postgis

https://gis.stackexchange.com/questions/338128/postgis-points-along-a-line-arent-actually-falling-on-the-line
This one contains a nice utlity function to segment a line by length, by using ST_LineSubstring.  Possible candidate for inclusion?

https://gis.stackexchange.com/questions/360670/how-to-break-a-linestring-in-n-parts-in-postgis

### Merge Lines That Don’t Touch
https://gis.stackexchange.com/questions/332780/merging-lines-that-dont-touch-in-postgis

Solution
No builtin function to do this, but one can be created in PL/pgSQL.


### Measure/4D relation querying within linestring using PostGIS
https://gis.stackexchange.com/questions/340689/measuring-4d-relation-querying-within-linestring-using-postgis

Solution
Uses DumpPoint and windowing functions

### Construct evenly-spaced points along a polygon boundary
https://gis.stackexchange.com/questions/360199/get-list-of-equidistant-points-on-polygon-border-postgis

### Find Segment of Line Closest to Point to allow Point Insertion
https://gis.stackexchange.com/questions/368479/finding-line-segment-of-point-on-linestring-using-postgis

Currently requires iteration.
Would be nice if the Linear Referencing functions could return segment index.
See https://trac.osgeo.org/postgis/ticket/892

```sql
CREATE OR REPLACE FUNCTION ST_LineLocateN( line geometry, pt geometry )
RETURNS integer
AS $$
    SELECT i FROM (
    SELECT i, ST_Distance(
        ST_MakeLine( ST_PointN( line, s.i ), ST_PointN( line, s.i+1 ) ),
        pt) AS dist
      FROM generate_series(1, ST_NumPoints( line )-1) AS s(i)
      ORDER BY dist
    ) AS t LIMIT 1;
$$
LANGUAGE sql STABLE STRICT;
```
### Insert LineString Vertices at Closest Point(s)
https://gis.stackexchange.com/questions/40622/how-to-add-vertices-to-existing-linestrings
https://gis.stackexchange.com/questions/370488/find-closest-index-in-line-string-to-insert-new-vertex-using-postgis
https://gis.stackexchange.com/questions/41162/adding-multiple-points-to-a-linestring-in-postgis

ST_Snap does this nicely
```sql
SELECT ST_AsText( ST_Snap('LINESTRING (0 0, 9 9, 20 20)',
  'MULTIPOINT( (1 1.1), (12 11.9) )', 0.2));
```
## Graphs
### Find Shortest Path through linear network
Input Parameters: linear network MultiLineString, start point, end point

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - perhaps: construct Convex Hull, take longest diameter, find shortest path between those points

https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi

## Temporal Trajectories

### Find Coincident Paths
https://www.cybertec-postgresql.com/en/intersecting-gps-tracks-to-identify-infected-individuals/

### Remove Stationary Points
https://gis.stackexchange.com/questions/290243/remove-points-where-user-was-stationary

## Input

### Parse Error loading OSM Polygons
https://gis.stackexchange.com/questions/346641/postgis-parse-error-invalid-geometry-after-using-st-multi-but-st-isvalid

#### Solution
Problem is incorrect order of columns, so trying to load an integer into a geometry field.
Better error messages would make this more obvious.

### Parse Error from non-WKT format text
https://gis.stackexchange.com/questions/311955/error-parse-error-invalid-geometry-postgis?noredirect=1&lq=1

## Output

### Generate GeoJSON Feature
https://gis.stackexchange.com/questions/112057/sql-query-to-have-a-complete-geojson-feature-from-postgis

