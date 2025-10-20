# Polygonal Coverages
{: .no_toc }

1. TOC
{:toc}

## Find Overlaps

**Solution**
Find the interection of pairs of polygons whose interiors intersect.

```sql
SELECT ST_CollectionExtract( ST_Intersection( a.geom, b.geom), 3))) AS overlap
FROM   polycov a
JOIN   polycov b
       ON ST_Intersects(a.geom, b.geom)
WHERE  ST_Relate(a.geom, b.geom, '2********')
       AND a.id > b.id
```

## Find Gaps

**Solution**
* Union the polygons in the coverage
* Find result union polygons containing holes
* Extract the holes as polygons

```sql
WITH union AS (
    SELECT (ST_Dump(ST_CoverageUnion(geom))).geom as geom
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

## Create trigger to enforce no overlaps
<https://gis.stackexchange.com/a/270802/14766>

Uses `ST_Relate(my_data.geom, g, '2********'))`.

```sql
CREATE TABLE my_data (
  id   int PRIMARY KEY,
  geom geometry
);

CREATE INDEX ON my_data USING gist(geom);

CREATE FUNCTION no_overlaps_in_my_data(id int, g geometry)
RETURNS boolean AS $$
SELECT NOT EXISTS (
  SELECT 1 FROM my_data
  WHERE my_data.id != id
    AND my_data.geom && g 
    AND ST_Relate(my_data.geom, g, '2********'));
$$ LANGUAGE sql;

ALTER TABLE my_data ADD CONSTRAINT no_overlaps CHECK (no_overlaps_in_my_data(id, geom));

```

Test:
```sql
INSERT INTO my_data VALUES (1, ST_Buffer(ST_MakePoint(1, 1), 1));
-- OK
INSERT INTO my_data VALUES (2, ST_Buffer(ST_MakePoint(3, 1), 1));
-- OK
INSERT INTO my_data VALUES (3, ST_Buffer(ST_MakePoint(2, 1), 1));
-- ERROR:  new row for relation "my_data" violates check constraint "no_overlaps"
```
