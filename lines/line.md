---
parent: Lines and Networks
---

# Line Processing
{: .no_toc }

1. TOC
{:toc}

## Select every Nth point from a LineString
https://stackoverflow.com/questions/60319473/postgis-how-do-i-select-every-second-point-from-linestring

## Extrapolate Lines
https://gis.stackexchange.com/questions/33055/extrapolating-a-line-in-postgis

ST_LineInterpolatePoint should be enhanced to allow fractions outside [0,1].

## Extend a LineString to the boundary of a polygon
https://gis.stackexchange.com/questions/345463/how-can-i-extend-a-linestring-to-the-edge-of-an-enclosing-polygon-in-postgis

![](https://i.stack.imgur.com/zvKKx.png)

#### Ideas for PostGIS
Create a new function `ST_LineExtract(line, index1, index2)` to extract a portion of a LineString between two indices

## Construct Line substring containing a set of Points
<https://gis.stackexchange.com/questions/408337/get-the-subline-of-a-linestring-consisting-of-all-intersecting-points-without-kn>

![](https://i.stack.imgur.com/LtZM0.png)

#### Solution 1 - Extract subline with new endpoints
* use `ST_LineLocatePoint()` on each intersection point to get the fraction of the point on the line
* use `ST_LineSubstring(my_line.geom, min(fraction), max(fraction))` with a GROUP BY on the line id to get subline

#### Solution 2 - Extract subline with all points added
```sql
SELECT  id, ST_Makeline(geom) AS geom
FROM   (
    SELECT  lns.id, 
            ST_LineSubstring(
                ln.geom,
                ST_LineLocatePoint(ln.geom, pts.geom),
                ST_LineLocatePoint(ln.geom, LEAD(pts.geom) OVER(ORDER BY pts.id))
            ) AS geom
    FROM    <line> AS ln
    CROSS JOIN
            <points> AS pts
) q
WHERE   geom IS NOT NULL
GROUP BY id;
```
