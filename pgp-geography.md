# Geography
{: .no_toc }

1. TOC
{:toc}

## Segmenting lines for accurate area computation
<https://gis.stackexchange.com/questions/206973/postgis-st-area-with-similar-geographies-producing-different-results>

The problem: calculating the area of a large polygon with straight lines that is converted to a `geography`.

Use ST_Segmentize to add intermediary segments in the lines.  This provides a better representation of the geodesic lines in geography.

```sql
SELECT
  ST_Area(
    ST_Segmentize(
      ST_Intersection(
        ST_GeomFromEWKT(
          'SRID=4326;POLYGON((-180 -52, -180 52, 180 52, 180 -52, -180 -52))'),
        geom
      ), 0.1
    )::geography
  ) AS area
FROM land;
```

## Find geometry with invalid geodetic coordinates
<https://gis.stackexchange.com/questions/411328/st-transform-error-latitude-or-longitude-exceeded-limits-14>

**Example**
```sql
SELECT ST_Transform(ST_SetSRID('POINT(575 90)'::geometry,4269),102008);

ERROR:  transform: latitude or longitude exceeded limits (-14)
```

**Find geodetic data with invalid coordinates** 
```sql
SELECT * FROM data_tbl
WHERE abs(ST_XMax(geom)) > 180 
   OR abs(ST_YMax(geom) > 90;
