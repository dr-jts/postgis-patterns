## Update, Delete
{: .no_toc }

1. TOC
{:toc}

## Update
### Update a column by a spatial condition
<https://gis.stackexchange.com/questions/364391/how-to-refer-to-another-table-in-a-case-when-statement-in-postgis>
```sql
UPDATE table1
    SET column3 = (
      SELECT 
        CASE
         WHEN table2.column7 >15 THEN 1
          ELSE 0
        END
      FROM table2 
      WHERE ST_INTERSECTS(table1.geom, table2.geom)
     --LIMIT 1
);
```
Is LIMIT 1 needed?

## Delete

### Delete Geometries Contained in Polygons
<https://gis.stackexchange.com/questions/372549/delete-lines-within-polygon>

Use an `EXISTS` expression.
`EXISTS` terminates the subquery as soon as a single row satisfying the `ST_Within` condition is found.
This is an efficient way when traversing a table by row (as in an `UPDATE`/`DELETE`), 
or otherwise comparing against a pre-selection (e.g. of ids).

```sql
DELETE FROM <geom_tbl> AS g
WHERE EXISTS (
  SELECT 1
  FROM   <poly_tbl> AS p
  WHERE  ST_Within(g.geom, p.geom)
);
```

### Delete Geometries NOT Contained in Polygons
<https://gis.stackexchange.com/questions/471587/delete-points-in-in-a-certain-area-using-postgis>

Use an `NOT EXISTS` expression.
`NOT EXISTS` terminates the subquery as soon as a single row satisfying the `ST_Within` condition is found.
This is an efficient way when traversing a table by row (as in an `UPDATE`/`DELETE`), 
or otherwise comparing against a pre-selection (e.g. of ids).

```sql
DELETE FROM <geom_tbl> AS g
WHERE NOT EXISTS (
  SELECT 1
  FROM   <poly_tbl> AS p
  WHERE  ST_Within(g.geom, p.geom)
);
```

### Delete Polygons intersecting Polygons in other tables
<https://stackoverflow.com/questions/71814571/postgis-delete-polygons-that-overlap>

Use `EXISTS` subqueries.

```sql
DELETE FROM <table_a> a
 WHERE EXISTS ( SELECT 1
                FROM <table_b> b
                WHERE ST_Intersects(a.geom, b.geom) )
    OR EXISTS ( SELECT 1
                FROM <table_c> c
                WHERE ST_Intersects(a.geom, c.geom) )
    OR EXISTS ( SELECT 1
                FROM <table_d> d
                WHERE ST_Intersects(a.geom, d.geom) );
```

### Delete features which have duplicates within distance
<https://gis.stackexchange.com/questions/356663/postgis-finding-duplicate-label-within-a-radius>

Features are duplicate if they have the same `value` column.

```sql
DELETE FROM features f
WHERE EXISTS (
    SELECT  1
    FROM    features
    WHERE   f.val = val AND f.id <> id AND ST_DWithin(f.geom, geom, <distance_in_CRS_units>)
);
```
