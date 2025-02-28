# Anti-Patterns
{: .no_toc }

1. TOC
{:toc}

## Using `ST_Distance` OR `ST_Intersects` with `ST_Buffer` for proximity queries

A simple way to test if geometries are near to (within a distance of) a given geometry is to compute the 
buffer of the query geometry and then use `ST_Intersects` against the buffer geometry.  However, this is somewhat inaccurate, since buffers are only approximations, and computing large buffers can be slow.

Another way is to use `ST_Distance`.  But this forces the full distance to be computed even though it is not needed.  Also, this query does not take advantage of spatial indexes.

Instead, use `ST_DWithin` instead, since it is automatically indexed, is faster to compute, and is more accurate. 
(it avoids having to compute the explicit buffer, which is only a close approximiation of the true distance).

## Using `ST_Contains/ST_Within` instead of `ST_Covers/ST_CoveredBy`

`ST_Contains` and `ST_Within` have a subtle quirk in their definition. It includes the requirement that at least one Interior point must be in common.  This means that "Polygons do not contain their boundary".  The result is that points and lines in the boundary of a polygon are not contained by/are not within the polygon.  `ST_Covers` and `ST_CoveredBy` do not have this semantic peculiarity (and may be faster to execute as well).

## Using `NOT IN` or `COUNT <...` instead of `NOT EXISTS`

This anti-pattern occurs when querying to find rows which are **not** in a given set of other rows 
(which may be in the same table or a different one).

Do not use `NOT IN subquery`, since it forces the subquery to compute the entire set of rows which do not match.

Instead, use the SQL *anti-join* pattern (see this [blog post](https://www.crunchydata.com/blog/rise-of-the-anti-join)).
This uses `NOT EXISTS subquery` to allow the query engine to short-circuit the subquery evaluation if a match is found. 

## Using `ST_MakePoint` or `ST_PointFromText` instead of `ST_Point`

`ST_Point` (and variants) is the standard function to use.  
`ST_MakePoint` is obsolete.
They are both much faster than `ST_PointFromText`.
