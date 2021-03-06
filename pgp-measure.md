# Measuring, Ordering
{: .no_toc }

1. TOC
{:toc}

## Measuring

### Find Median width of Road Polygons
<https://gis.stackexchange.com/questions/364173/calculate-median-width-of-road-polygons>

### Unbuffering - find average distance from a buffer and source polygon
<https://gis.stackexchange.com/questions/33312/is-there-a-st-buffer-inverse-function-that-returns-a-width-estimation>

Also: <https://gis.stackexchange.com/questions/20279/calculating-average-width-of-polygon>

### Compute Length and Width of an arbitrary rectangle
<https://gis.stackexchange.com/questions/366832/get-dimension-of-rectangular-polygon-postgis>


## Ordering Geometry

### Ordering a Square Grid
<https://gis.stackexchange.com/questions/346519/sorting-polygons-into-a-n-x-n-spatial-array>

<https://gis.stackexchange.com/questions/255512/automatically-name-rectangles-by-spatial-order-or-position-to-each-other>

### Serpentine Ordering
<https://gis.stackexchange.com/questions/176197/seeking-tool-algorithm-for-assigning-code-to-enumeration-areas-polygons-using>

No solution in the post

Also
<https://gis.stackexchange.com/questions/73978/numbering-polygons-according-to-their-spatial-relationships>

### Ordering Polygons along a line
<https://gis.stackexchange.com/questions/201306/numbering-adjacent-polygons-in-sequential-order>

No explicit solution given, but suggestion is to compute adjacency graph and then do a graph traversal

### Order polygons touched by a Line
<https://gis.stackexchange.com/questions/317401/maintaining-order-and-repetition-of-cell-names-using-postgis>

### Connecting Circles Into a Polygonal Path
<https://gis.stackexchange.com/questions/246521/connecting-circles-with-lines-cover-all-circles-using-postgis>

### Ordered list of polygons intersecting a line
<https://gis.stackexchange.com/questions/179061/find-all-intersections-of-a-linestring-and-a-polygon-and-the-order-in-which-it-i>

### Ordering points along lines

Given:
* way (id, name, geom (multistring))
* station (id, name, geom (polygon)) - circles for each station, which may or may not intersect way line

Find the station order along each way

<https://gis.stackexchange.com/questions/379220/finding-station-order-number-along-way-using-postgis>

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
