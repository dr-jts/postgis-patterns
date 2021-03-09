
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

## Output

### Generate GeoJSON Feature
https://gis.stackexchange.com/questions/112057/sql-query-to-have-a-complete-geojson-feature-from-postgis
