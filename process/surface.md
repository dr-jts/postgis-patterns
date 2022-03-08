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
https://gis.stackexchange.com/questions/373153/spatial-interpolation-in-postgis-without-outputting-raster

```sql
UPDATE <points> AS itp
  SET  "Z" = (
    SELECT SUM(z/d)/SUM(1/d)
    FROM   (
      SELECT smpl."Z" as z,
             ST_Distance(itp.geom, smpl.geom)^<P-value> AS d
      FROM   <point> AS smpl
      ORDER BY
             itp.geom <-> smpl.geom
      WHERE  smpl."Z" IS NOT NULL
      LIMIT  <sample_size>
    ) sq
    WHERE  ipt."Z" IS NULL
  ) q;
```

### Compute Z value of a TIN at a Point
<https://gis.stackexchange.com/questions/237506/query-z-at-intersecting-point-locations-from-a-multipolygonz>

![](https://i.stack.imgur.com/DST02.png)

**Solution**

* Use `ST_3DIntersection` (code [here](https://gis.stackexchange.com/a/332906/14766) - requires SF_CGAL)
* Use a custom function (code [here](https://gis.stackexchange.com/a/238297/14766))

#### PostGIS TO DO - Add ST_3DInterpolate ?
