
# Input/Output
{: .no_toc }

1. TOC
{:toc}

## Input

### Parse Error loading OSM Polygons
https://gis.stackexchange.com/questions/346641/postgis-parse-error-invalid-geometry-after-using-st-multi-but-st-isvalid

#### Solution
Problem is incorrect order of columns, so trying to load an integer into a geometry field.
Better error messages would make this more obvious.

### Parse Error from non-WKT format text
https://gis.stackexchange.com/questions/311955/error-parse-error-invalid-geometry-postgis?noredirect=1&lq=1

## GeoJSON

### Generate GeoJSON Feature
<https://gis.stackexchange.com/questions/112057/sql-query-to-have-a-complete-geojson-feature-from-postgis>

```sql
SELECT jsonb_build_object(
    'type',       'Feature',
    'id',         gid,
    'geometry',   ST_AsGeoJSON(geom)::jsonb,
    'properties', to_jsonb(row) - 'gid' - 'geom'
) FROM (SELECT * FROM input_table) row;
```

### Generate GeoJSON FeatureCollection
<https://gis.stackexchange.com/questions/112057/sql-query-to-have-a-complete-geojson-feature-from-postgis>

```sql
SELECT jsonb_build_object(
    'type',     'FeatureCollection',
    'features', jsonb_agg(features.feature)
)
FROM (
  SELECT jsonb_build_object(
    'type',       'Feature',
    'id',         gid,
    'geometry',   ST_AsGeoJSON(geom)::jsonb,
    'properties', to_jsonb(inputs) - 'gid' - 'geom'
  ) AS feature
  FROM (SELECT * FROM input_table) inputs) features;
  ```

### Generate GeoJSON Features for geometry bounds
<https://gis.stackexchange.com/questions/398394/get-bounds-as-geojson>

```sql
SELECT JSONB_BUILD_OBJECT(
         'type',     'FeatureCollection',
         'features', JSONB_AGG(ST_AsGeoJSON(q.*)::JSONB)
       ) AS fc
FROM   (
    SELECT 1 AS id,
           ST_Envelope(ST_Expand('POINT(0 0)'::GEOMETRY, 1)) AS geom
) q;
```

### Custom aggregate functions for GeoJSON FeatureCollections
<https://gist.github.com/geozelot/b2e5cd65dd7f85ec381aeee14e0149ae>
```sql
SELECT ST_AsFeatureCollection(q.*) AS geojson
FROM   (
    SELECT 1 AS id,
           ST_Envelope(ST_Expand('POINT(0 0)'::GEOMETRY, 1)) AS geom
) q;
```
