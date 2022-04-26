---
parent: Lines and Networks
---

# Temporal Trajectories
{: .no_toc }

1. TOC
{:toc}

## Find Coincident Paths
<https://www.cybertec-postgresql.com/en/intersecting-gps-tracks-to-identify-infected-individuals/>

![](https://www.cybertec-postgresql.com/wp-content/uploads/2020/04/GPS-Tracking5.jpg)

## Remove Stationary Points
<https://gis.stackexchange.com/questions/290243/remove-points-where-user-was-stationary>

## Find Path durations within Polygons
<https://gis.stackexchange.com/questions/362537/summarise-duration-of-points-within-polygon>

Given:
* a table of points along GPS tracks
* a table of polygons

Finds the path duration within each polygon.

```sql
WITH trajectory AS (
    SELECT  track,
            ST_SetSRID(ST_MakeLine(ST_MakePointM(ST_X(geom), ST_Y(geom), EXTRACT(EPOCH FROM ts)) ORDER BY ts),4326) AS geom
    FROM    gps
    GROUP BY track
)
SELECT  p.id,
        t.track,
        SUM( ST_InterpolatePoint(b.geom, ST_EndPoint(dmp.geom)) - ST_InterpolatePoint(b.geom, ST_StartPoint(dmp.geom)) )
FROM    poly AS p
JOIN    trajectory AS t
  ON    ST_Intersects(p.geom, t.geom),
        LATERAL ST_Dump(ST_Intersection(p.geom, t.geom)) AS dmp
GROUP BY
        p.id, t.track;
```
**Issues**
* May not work if paths enter and exit polygon at same points?
