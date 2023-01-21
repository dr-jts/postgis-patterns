---
parent: Processing
---

# Constructions
{: .no_toc }

1. TOC
{:toc}

## Geometric Shapes

### Construct regular N-gons
<https://gis.stackexchange.com/questions/426933/buffer-with-postgis-in-the-form-of-a-pentagon-or-hexagon>

**Solution 1: SQL**

Construct a pentagon (5 sides) of radius 2 centred at (10,10). Those values can be changed to whatever is needed.

```sql
SELECT ST_MakePolygon( ST_MakeLine( ARRAY_AGG( 
    ST_Point( 10 + 2 * COSD(90 + i * 360 / 5 ), 
              10 + 2 * SIND(90 + i * 360 / 5 )  ))))
  FROM generate_series(0, 5) AS s(i);
```

**Solution 2 - Function `ST_MakeNGon`**

<https://gist.github.com/geozelot/ddf88a9ae0438d7a46f176e9555ce7a1>
```sql
/*
 * @in_params
 * center - center POINT geometry
 * radius - circumradius [in CRS units] (cirlce that inscribes the *gon)
 * sides  - desired side count (e.g. 6 for Hexagon)
 * rot    - rotation offset, clockwise [in degree]; DEFAULT 0.0 (corresponds to planar NORTH)
 *
 * @out_params
 * ngon   - resulting N-gon POLYGON geometry
 *
 *
 * The function will create a @sides sided regular, equilateral & equiangular Polygon
 * from a given @center and @radius and optional @rot rotation from NORTH
 */
 
CREATE OR REPLACE FUNCTION ST_MakeNGon(
  IN  center   GEOMETRY(POINT),
  IN  radius   FLOAT8,
  IN  sides    INT,
  IN  rot      FLOAT8 DEFAULT 0.0,
  OUT ngon     GEOMETRY(POLYGON)
) LANGUAGE 'plpgsql' IMMUTABLE STRICT PARALLEL SAFE AS
  $$
  DECLARE
    _x FLOAT8  := ST_X($1);
    _y FLOAT8  := ST_Y($1);
	
    _cr FLOAT8 := 360.0/$3;
		
    __v GEOMETRY(POINT)[];
	
  BEGIN
    FOR i IN 0..$3 LOOP
      __v[i] := ST_MakePoint(_x + $2*SIND(i*_cr + $4), _y + $2*COSD(i*_cr + $4));
    END LOOP;
		
    ngon := ST_MakePolygon(ST_MakeLine(__v));
  END;
  $$
;
```

### Construct an envelope for a Geometry at an Angle
<https://gis.stackexchange.com/questions/443727/how-to-control-the-angle-of-st-orientedenvelope-postgis>

**Solution**
* Rotate the geometry CW to the desired angle
* Compute the envelope
* Rotate the envelope CCW by the desired angle

```sql
WITH data(angle, geom) AS (VALUES
   (0.2, 'POLYGON ((10 60, 10 20, 50 30, 70 10, 90 60, 60 80, 40 60, 30 90, 10 60))'::geometry)
)
SELECT ST_Rotate( ST_Envelope(
            ST_Rotate(geom, -angle, ST_Centroid(geom))),
        angle, ST_Centroid(geom)) AS envRot
    FROM data;
```

## Geodetic Shapes

### Construct ellipses in WGS84
<https://gis.stackexchange.com/questions/218159/postgis-ellipse-issue>

<https://gis.stackexchange.com/questions/409129/making-ellipses-on-a-global-data-set>

## 3D Shapes

### 3D Centroid
<https://gis.stackexchange.com/questions/449451/3d-centroid-in-postgis>

```
WITH pts AS
(SELECT 
  geom(
    ST_DumpPoints(
      ST_RemovePoint(
        ST_ExteriorRing(
          ST_GeomFromText('POLYGON Z ((0 0 0, 1 0 0, 1 0 1, 1 1 1, 0 1 1, 0 0 1, 0 0 0 ))')
        ), 0 -- remove first point because, for a polygon, first point = last point by definition
      )
    )
  )
)
SELECT 
  ST_AsText(
    ST_MakePoint(
      AVG(ST_X(geom)),
      AVG(ST_Y(geom)),
      AVG(ST_Z(geom))
    )
  ) geom
FROM pts;
```
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

Also: <https://gis.stackexchange.com/questions/316000/using-st-union-to-combine-several-polygons-to-one-multipolygon-using-postgis>

**Suggestions**

* Buffer positively and negatively (see `ST_BufferedUnion`in [PostGIS Addons](https://github.com/pedrogit/postgisaddons/blob/master/postgis_addons.sql) )
* [Answer](https://gis.stackexchange.com/a/316124/14766) using `ST_Buffer(geom, dist, 'join=mitre mitre_limit=5.0')` with pos/neg distance

### Construct polygon joining two polygons
<https://gis.stackexchange.com/questions/352884/how-can-i-get-a-polygon-of-everything-between-two-polygons-in-postgis>

![](https://i.stack.imgur.com/a7idE.png)

**See Also**
<https://gis.stackexchange.com/questions/49726/postgis-algorithm-to-unite-points-of-two-geometries-that-are-within-specified-ra>

**Solution**
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

### Extend Polygon to meet Line
<https://gis.stackexchange.com/questions/408202/postgis-st-convexhull-but-perpendicular-to-a-line>

![](https://i.stack.imgur.com/DQw7k.png)

**Solution Outline**

* For each vertex in the polygon shell, use `ST_LineLocatePoint` to find its fractional index along the line.
* Construct the line segments from each vertex to the line, and pick the shortest ones with max and min index
* Construct the subline along the baseline between the max and min indices
* Extract all the line segments from the polygon
* Polygonize the extracted and constructed line segments. This should produce two polygons
* Union the polygonization results

## Grids

### Construct Land-Constrained Point Grid
<https://korban.net/posts/postgres/2019-10-17-generating-land-constrained-point-grids/>

![](https://korban.net/img/2019-10-16-12-44-48.png)

### Construct Square Grid
<https://gis.stackexchange.com/questions/16374/creating-regular-polygon-grid-in-postgis>

![](https://i.stack.imgur.com/vQY0Z.png)

<https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis>

<https://gis.stackexchange.com/questions/271234/creating-a-grid-on-a-polygon-and-find-number-of-points-in-each-grid>

### Construct Quadrilateral Grid
Construct a regularly spaced grid in an arbitrary quadrilateral.

<https://gis.stackexchange.com/questions/437745/creating-an-irregular-point-grid-with-provided-row-and-column-numbers-in-postgis>

![](https://i.stack.imgur.com/vwqg0.png)

**Solution**
Use a pseudo-projective transformation of a grid on a unit square to the quadrilateral.
Transformation is a simple non-linear combination of three basis vectors.
Formula and explanation are [here](https://math.stackexchange.com/a/863702).

![](https://i.stack.imgur.com/fTMhb.png)

```sql
WITH quad AS (SELECT 
  5 AS LLx, 5 AS LLy,
  10 AS ULx, 30 AS ULy,
  25 AS URx, 25 AS URy,
  30 AS LRx, 0 AS LRy
),
vec AS (
  SELECT  LLx AS ox, LLy AS oy,
          ULx - LLx AS ux, ULy - LLy AS uy, 
          LRx - LLx AS vx, LRy - LLy As vy, 
          URx - LLx - ((ULX - LLx) + (LRx - LLx)) AS wx, 
          URy - LLy - ((ULy - LLy) + (LRy - LLy)) AS wy 
  FROM quad
),
grid AS (SELECT x / 10.0 AS x, y / 10.0 AS y 
  FROM        generate_series(0, 10) AS sx(x)
  CROSS JOIN  generate_series(0, 10) AS sy(y)
)
SELECT ST_Point(ox + ux * x + vx * y + wx * x * y, 
                oy + uy * x + vy * y + wy * x * y) AS geom
  FROM vec CROSS JOIN grid;
  ```
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

### Construct all diagonals in a concave polygon
<https://gis.stackexchange.com/questions/407946/performant-possible-pairs-in-concave-polygon>

![](https://i.stack.imgur.com/TsP6P.png)

**Solution**

Using only polygon shell:
```sql
WITH poly AS 
(SELECT ST_MakePolygon(ST_ExteriorRing((ST_Dump(geom)).geom)) geom FROM
    (SELECT ST_GeomFromText('POLYGON((25.390624999999982 23.62298461759423,18.183593749999982 19.371888927008566,7.812499999999982 17.87273879517762,5.878906249999982 24.90497143578641,9.570312499999982 25.223427998254586,12.734374999999982 25.064303191014304,15.195312499999982 30.048744443788348,22.578124999999982 30.352588399664125,24.687499999999982 25.857838723065772,25.390624999999982 23.62298461759423))') AS geom
) boo),
poly_ext AS (SELECT ST_ExteriorRing((ST_Dump(geom)).geom) geom FROM poly),
points AS (SELECT (g.gdump).path AS id, (g.gdump).geom AS geom
                  FROM (SELECT ST_DumpPoints(geom) AS gdump FROM poly_ext) AS g)
SELECT ST_MakeLine(a.geom, b.geom) AS geom FROM points a CROSS JOIN points b
       JOIN poly ON ST_Contains(poly.geom, ST_MakeLine(a.geom, b.geom)) WHERE a.id < b.id;
```

### Construct longest horizontal line within polygon
<https://gis.stackexchange.com/questions/32552/calculating-maximum-distance-within-polygon-in-x-direction-east-west-direction>

**Algorithm**
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




