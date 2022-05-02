# Polygonal Coverages
{: .no_toc }

1. TOC
{:toc}

## Find Overlaps

**Solution**
Find pairs of polygons whose interiors intersect.

```sql
SELECT ST_CollectionExtract( ST_Intersection( a.geom, b.geom), 3))) AS overlap
FROM   polycov a
JOIN   polycov b
       ON ST_Intersects(a.geom, b.geom)
WHERE  ST_Relate(a.geom, b.geom, '2********')
       AND a.id > b.id
```

## Find Gaps/Slivers

**Solution**
* Union the polygons in the coverage
* Find result union polygons containing holes
* Extract the holes as polygons

```sql
WITH union AS (
    SELECT (ST_DUMP(ST_Union(geom))).geom as geom
        FROM polycov As f 
),
hasgaps AS (
    SELECT geom 
    FROM union
    WHERE ST_NumInteriorRings(geom) > 0
),
SELECT ST_CollectionExtract(ST_BuildArea(ST_InteriorRingN(geom, i)), 3) as gap
FROM hasgaps
CROSS JOIN generate_series(1, ST_NumInteriorRings(geom)) as i
```
