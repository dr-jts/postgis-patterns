# Anti-Patterns
{: .no_toc }

1. TOC
{:toc}

## Using `ST_Intersects/ST_Buffer` for proximity queries

Use `ST_DWithin` instead, since it is automatically indexed and is faster to compute.

## Using `ST_Contains/ST_Within` instead of`ST_Covers/ST_CoveredBy`

`ST_Contains` and `ST_Within` have a subtle quirk in their definition. It includes the requirement that at least one Interior point must be in common.  This means that "Polygons do not contain their boundary".  The result is that points and lines in the boundary of a polygon are not contained by/are not within the polygon.  `ST_Covers` and `ST_CoveredBy` do not have this semantic peculiarity (and may be faster to execute as well).
