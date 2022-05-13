---
parent: Processing
---

# Contouring and Surface Interpolation
{: .no_toc }

1. TOC
{:toc}

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
<https://gis.stackexchange.com/questions/373153/spatial-interpolation-in-postgis-without-outputting-raster>

Use [Inverse-Distance-Weighting](https://en.wikipedia.org/wiki/Inverse_distance_weighting) (IDW) interpolation on a subset of nearby points.
The **power parameter** `POWER` is chosen according to the weighting desired.. 

```sql
SELECT SUM( z/d ) / SUM( 1/d ) AS z
FROM (
  SELECT smpl.z as z,
         ST_Distance(itp.geom, smpl.geom) ^ POWER AS d
  FROM   <point> AS smpl
  ORDER BY
         itp.geom <-> smpl.geom
  WHERE  smpl.Z IS NOT NULL
  LIMIT  <sample_size>
) t;
```

### Compute Z value of a TIN at a Point
<https://gis.stackexchange.com/questions/237506/query-z-at-intersecting-point-locations-from-a-multipolygonz>

![](https://i.stack.imgur.com/DST02.png)

**Solution**

* Use `ST_3DIntersection` (code [here](https://gis.stackexchange.com/a/332906/14766) - requires SF_CGAL)
* Use a custom function (code [here](https://gis.stackexchange.com/a/238297/14766))

#### PostGIS TO DO - Add ST_3DInterpolate ?
