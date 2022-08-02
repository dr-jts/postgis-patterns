---
parent: Overlay
---

# Lines
{: .no_toc }

1. TOC
{:toc}

## Line Intersection
### Intersection of Lines which are not exactly coincident
<https://stackoverflow.com/questions/60298412/wrong-result-using-st-intersection-with-postgis/60306404#60306404>


## Line Merging
### Merge/Node Lines in a linear network
<https://gis.stackexchange.com/questions/238329/how-to-break-multilinestring-into-constituent-linestrings-in-postgis>

**Solution**
Use ST_Node, then ST_LineMerge

### Merge Lines and preserve direction
<https://gis.stackexchange.com/questions/353565/how-to-join-linestrings-without-reversing-directions-in-postgis>

SOLUTION 1
Use a recursive CTE to group contiguous lines so they can be merged

```sql
WITH RECURSIVE
data AS (SELECT
--'MULTILINESTRING((0 0, 1 1), (2 2, 1 1), (2 2, 3 3), (3 3, 4 4))'::geometry
'MULTILINESTRING( (0 0, 1 1), (1 1, 2 2), (3 3, 2 2), (4 4, 3 3), (4 4, 5 5), (5 5, 6 6) )'::geometry
 AS geom)
,lines AS (SELECT t.path[1] AS id, t.geom FROM data, LATERAL ST_Dump(data.geom) AS t)
,paths AS (
  SELECT * FROM
    (SELECT l1.id, l1.geom, l1.id AS startid, l2.id AS previd
      FROM lines AS l1 LEFT JOIN lines AS l2 ON ST_EndPoint(l2.geom) = ST_StartPoint(l1.geom)) AS t
    WHERE previd IS NULL
  UNION ALL
  SELECT l1.id, l1.geom, startid, p.id AS previd
    FROM paths p
    INNER JOIN lines l1 ON ST_EndPoint(p.geom) = ST_StartPoint(l1.geom)
)
SELECT ST_AsText( ST_LineMerge(ST_Collect(geom)) ) AS geom
  FROM paths
  GROUP BY startid;
```

SOLUTION 2
ST_LIneMerge merges lines irrespective of direction.  
Perhaps a flag could be added to respect direction?

See Also
<https://gis.stackexchange.com/questions/74119/a-linestring-merger-algorithm>

### Merge lines to simplify a road network
Merge lines with common attributes at degree-2 nodes

<https://gis.stackexchange.com/questions/326433/st-linemerge-to-simplify-road-network>


## Line Splitting

### Split Self-Overlapping Lines at Points not on the lines
<https://gis.stackexchange.com/questions/347790/splitting-self-overlapping-lines-with-points-using-postgis>

### Split Lines by Polygons
<https://gis.stackexchange.com/questions/215886/split-lines-by-polygons>

### Split Lines by Polygons and order the sections
<https://gis.stackexchange.com/questions/437163/splitting-lines-by-multiple-polygons-taking-length-of-particular-segment-using>

![](https://i.stack.imgur.com/ren10.png)

**Solution**

* Use `ST_Dump` to extract all sections if the intersection is a `MultiLineString`
* Order the line sections by the location of their start point along the parent line

```sql
WITH polys(id, geom) AS (VALUES
  (1,'POLYGON((2 2, 2 4, 4 4, 4 2, 2 2))'::geometry),
  (2,'POLYGON((4 2, 4 4, 5 4, 5 2, 4 2))'::geometry),
  (3,'POLYGON((5 2, 5 4, 5.5 4, 5.5 2, 5 2))'::geometry),
  (4,'POLYGON((5.5 2, 5.5 4, 7 4, 7 2, 5.5 2))'::geometry),

  (5,'POLYGON((3 9, 4 9, 4 6, 6 6, 6 9, 7 9, 7 5, 3 5, 3 9))'::geometry),
  (6,'POLYGON((1 9, 3 9, 3 5, 1 5, 1 9))'::geometry),
  (7,'POLYGON((4 9, 6 9, 6 6, 4 6, 4 9))'::geometry),
  (8,'POLYGON((7 9, 8 9, 8 5, 7 5, 7 9))'::geometry)
),
lines(id, geom) AS (VALUES
  (1, 'LINESTRING(3 3, 6 3)'::geometry),
  (2, 'LINESTRING(2 7, 9 7)'::geometry)
),
sections AS (SELECT p.id polyid, 
            l.id lineid, 
            l.geom as linegeom, 
            (ST_Dump( ST_Intersection(p.geom, l.geom) )).geom section
    FROM polys p
    JOIN lines l
    ON ST_Intersects(p.geom, l.geom)
)
SELECT lineid, polyid,
  row_number() OVER( PARTITION BY lineid 
                     ORDER BY ST_LineLocatePoint(linegeom, ST_StartPoint(section)) ) sectioNum, 
  ST_Length(section) length, 
  ST_AsText(section) 
FROM sections;
```

## Overlay - Lines
<https://gis.stackexchange.com/questions/186242/how-to-get-smallest-line-segments-from-intersection-difference-of-multiple-ove>

### Count All Intersections Between 2 Linestrings
<https://gis.stackexchange.com/questions/347790/splitting-self-overlapping-lines-with-points-using-postgis>

### Remove Line Overlaps Hierarchically
<https://gis.stackexchange.com/questions/372572/how-to-remove-line-overlap-hierachically-in-postgis-with-st-difference>
