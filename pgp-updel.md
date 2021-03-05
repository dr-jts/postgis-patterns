## PostGIS Patterns - Update, Delete

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
### Delete Lines Contained in Polygons
https://gis.stackexchange.com/questions/372549/delete-lines-within-polygon

Use an EXISTS expression:
```sql
DELETE
FROM   <lines> AS ln
WHERE  EXISTS (
  SELECT 1
  FROM   <poly> AS pl
  WHERE  ST_Within(ln.geom, pl.geom)
);
```
If the ST_Within check hits the first TRUE (selecting a truthy 1), the sub-query terminates for the current row (no matter if there were more than one hit).

This is among the most efficient ways for when a table has to be traversed by row (as in an UPDATE/DELETE), or otherwise compared against a pre-selection (of e.g. ids).
