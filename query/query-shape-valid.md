---
parent: Querying
---

# Shape, Validity, Duplicates

{: .no_toc }

1. TOC
{:toc}

## Query Geometric Shape

### Find narrow Polygons
https://gis.stackexchange.com/questions/316128/identifying-long-and-narrow-polygons-in-with-postgis

#### Solution 1 - Radius of Maximum Inscribed Circle
Use function [`ST_MaximumInscribedCircle`](https://postgis.net/docs/manual-dev/ST_MaximumInscribedCircle.html)
to compute the radius of the Maximum Inscribed Circle of each polygon, and then select using desired cut-off.
This is much faster than computing negative buffers.

#### Solution 2 - Negative Buffer
Detect thin polygons using a test: `ST_Area(ST_Buffer(geom, -10)) = 0`.
This is not very performant.

#### Solution 3 - Thinness Ratio
Use the Thinness Ratio:  `TR(Area,Perimeter) = Area * 4 * pi / (Perimter^2)`.

This is only a heuristic approximation, and it's hard to choose appropriate cut-off for the value of the ratio.

See https://gis.stackexchange.com/questions/151939/explanation-of-the-thinness-ratio-formula



## Query Invalid Geometry

### Skip invalid geometries when querying
https://gis.stackexchange.com/questions/238748/compare-only-valid-polygon-geometries-in-postgis

## Query Duplicates
### Find and Remove duplicate geometry rows
https://gis.stackexchange.com/questions/124583/delete-duplicate-geometry-in-postgis-tables
