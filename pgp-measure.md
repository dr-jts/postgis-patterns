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

### Convert distance to geodetic degrees
<https://gis.stackexchange.com/questions/430023/converting-1-meter-to-degrees-at-specific-location-using-postgis>

**Solution**

Geodetic distance depends on the latitude of the initial point and the azimuth (bearing).

```sql
WITH pt AS (SELECT 5 AS long, 40 AS lat)
SELECT 
    1 / ST_Distance(
        ST_Point(pt.long, pt.lat)::geography,
        ST_Point(pt.long, pt.lat + 1)::geography
    ) AS one_meter_lat_north,
    1 / ST_Distance(
        ST_Point(pt.long, pt.lat)::geography,
        ST_Point(pt.long + 1, pt.lat)::geography
    ) AS one_meter_long_east
FROM pt;
```

**Solution 2**
<https://gist.github.com/geozelot/5cf8425160b98394d0f9ba50ce76917e>
```sql
/*
 * Returns a close approximation of distance in degrees of a sphere at given @latitude that corresponds to
 * @distance meter surface distance, in the direction of north based @angle in degree, from the center of the ellipse.
 * Angularity defaults to 90.0 (eastward) and shortcuts execution - corresponds to the simple factor of
 * reduced surface distance per degree of longitude at increasing latitudes.
 */

CREATE OR REPLACE FUNCTION ST_ToGeodeticDegrees(
  latitude FLOAT,
  distance FLOAT,
  angle    FLOAT DEFAULT 90.0
) RETURNS FLOAT IMMUTABLE STRICT PARALLEL SAFE
  LANGUAGE 'plpgsql' AS
  $BODY$
  DECLARE
    ld FLOAT := $2 / DEGREE(6371000.0 * COS(RADIANS($1)))

  BEGIN
    IF $3 IN (90.0, 180.0) THEN
      RETURN ld;
    END IF;
	
    RETURN SQRT(POW(ld, 2) * POW(COS(RADIANS($3)), 2) + POW(SIN(RADIANS($3)), 2));
  END;
  $BODY$
;
```

### Convert bearing to Cardinal Direction
* <https://gis.stackexchange.com/a/425534/14766>
* <https://trac.osgeo.org/postgis/wiki/UsersWikiCardinalDirection>

Converts a bearing from `ST_Azimuth` to a cardinal direction (N, NW, W, SW, S, SE, E, or NE).

```sql
CREATE OR REPLACE FUNCTION ST_CardinalDirection(azimuth float8) RETURNS character varying AS
$BODY$SELECT CASE
  WHEN $1 < 0.0 THEN 'less than 0'
  WHEN degrees($1) < 22.5 THEN 'N'
  WHEN degrees($1) < 67.5 THEN 'NE'
  WHEN degrees($1) < 112.5 THEN 'E'
  WHEN degrees($1) < 157.5 THEN 'SE'
  WHEN degrees($1) < 202.5 THEN 'S'
  WHEN degrees($1) < 247.5 THEN 'SW'
  WHEN degrees($1) < 292.5 THEN 'W'
  WHEN degrees($1) < 337.5 THEN 'NW'
  WHEN degrees($1) <= 360.0 THEN 'N'
END;$BODY$ LANGUAGE sql IMMUTABLE COST 100;
COMMENT ON FUNCTION ST_CardinalDirection(float8) IS 'input azimuth in radians; returns N, NW, W, SW, S, SE, E, or NE';
```

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
