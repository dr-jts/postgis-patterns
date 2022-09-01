---
parent: Querying
---

# Spatial Statistics
{: .no_toc }

1. TOC
{:toc}

## Count Points contained in Polygons

#### Solution - LATERAL
Good use case for JOIN LATERAL
```sql
SELECT bg.geoid, bg.geom,  bg.total_pop AS total_population, 
       bg.med_inc AS median_income,
       t.numbirds
  FROM bg_pop_income bg
  JOIN LATERAL
        (SELECT COUNT(1) as numbirds 
           FROM  bird_loc bl 
           WHERE ST_within(bl.loc, bg.geom)) AS t ON true;
```
#### Solution using GROUP BY
Almost certainly less performant
```sql
SELECT bg.geoid, bg.geom, bg.total_pop, bg.med_inc, 
       COUNT(bl.global_unique_identifier) AS num_bird_counts
  FROM  bg_pop_income bg 
  LEFT OUTER JOIN bird_loc bl ON ST_Contains(bg.geom, bl.loc)
  GROUP BY bg.geoid, bg.geom, bg.total_pop, bg.med_inc;
```
## Count points from two tables which lie inside polygons
https://stackoverflow.com/questions/59989829/count-number-of-points-from-two-datasets-contained-by-neighbourhood-polygons-u

## Count polygons which lie within two layers of polygons
https://gis.stackexchange.com/questions/115881/how-many-a-features-that-fall-both-in-the-b-polygons-and-c-polygons-are-for-each

***(Question is for ArcGIS; would be interesting to provide a PostGIS answer)***

## Count number of adjacent polygons in a coverage
<https://gis.stackexchange.com/questions/389448/postgis-count-adjacent-polygons-connected-by-line>
(First query)
```sql
SELECT a.id, a.geom, a.codistat, a.name, num_adj
FROM municipal a 
JOIN LATERAL (SELECT COUNT(1) num_adj 
              FROM municipal b
              WHERE ST_Intersects(a.geom, b.geom)
              ) t ON true;
```

## Count number of adjacent polygons in a coverage connected by a line
<https://gis.stackexchange.com/questions/389448/postgis-count-adjacent-polygons-connected-by-line>
```sql
SELECT a.id, a.geom, a.codistat, a.name, num_adj
FROM municipal a 
JOIN LATERAL (SELECT COUNT(1) num_adj 
              FROM municipal b
              JOIN way w
              ON ST_Intersects(b.geom, w.geom)
              WHERE ST_Intersects(a.geom, b.geom)
                 AND ST_Intersects(a.geom, w.geom)
              ) t ON true;
```

## Find Median of values in a Polygon neighbourhood in a Polygonal Coverage
<https://gis.stackexchange.com/questions/349251/finding-median-of-polygons-that-share-boundaries>

## Sum length of Lines intersecting Polygons
```sql
SELECT county, SUM(ST_Length(ST_Intersection(counties.geom,routes.geom)))
FROM counties
JOIN routes ON ST_Intersects(counties.geom, routes.geom)
GROUP BY county;
```
See following (but answers are not great)
<https://gis.stackexchange.com/questions/143438/calculating-total-line-lengths-within-polygon>


## Find geometry with maximum value in groups
<https://gis.stackexchange.com/questions/408162/get-max-value-from-points-based-on-distinct-id-and-keep-geometry-for-the-new-se>

From a table containing a group_id, a value and a geometry, find the record in each group which has the maximum value in the group.

NOTE: This is a classic SQL problem, not unique to spatial data.

### Using ROW_NUMBER window function
```sql
SELECT id, val, geom
FROM   (
  SELECT *, ROW_NUMBER() OVER ( PARTITION BY id ORDER BY val DESC ) AS _rank
  FROM   tbl
) q
WHERE  _rank = 1;
```
### Using self-join
```sql
SELECT a.id, MAX( a.val ), b.geom
FROM tbl a 
GROUP BY id
LEFT JOIN tbl b ON a.id, b.id;
```

## Find extremal points of Polygons
<https://gis.stackexchange.com/questions/323289/how-to-calculate-extreme-point-most-north-or-east-etc-of-geographypolygon-ty>

Extract unique points of polygons using `ST_DumpPoints(ST_RemoveRepeatedPoints(ST_Points`, then use `RANK` window functions 
to extract points with maximmal/minimal X and Y.

**Note:** it might be more efficient to use the `path` information from `ST_DumpPoints` to eliminate the duplicate start and end point.

```sql
WITH data(id, geom) AS (VALUES
   (1, 'POLYGON((-71.1776585052917 42.3902909739571,-71.1776820268866 42.3903701743239, -71.1776063012595 42.3903825660754,-71.1775826583081 42.3903033653531,-71.1776585052917 42.3902909739571))'::geometry)
  ,(2, 'POLYGON ((-71.1775 42.3902, -71.1773 42.3908, -71.1769 42.3905, -71.177 42.39, -71.1775 42.3902))'::geometry)
),
pts AS (SELECT id, (ST_DumpPoints(ST_RemoveRepeatedPoints(ST_Points(geom)))).geom FROM data),
rank AS (SELECT id, ST_AsText(geom), 
  RANK() OVER (PARTITION BY id ORDER BY ST_X(geom) DESC) AS rank_max_x,
  RANK() OVER (PARTITION BY id ORDER BY ST_X(geom) ASC)  AS rank_min_x,
  RANK() OVER (PARTITION BY id ORDER BY ST_Y(geom) DESC) AS rank_max_y,
  RANK() OVER (PARTITION BY id ORDER BY ST_Y(geom) ASC)  AS rank_min_y
  FROM pts
)
SELECT * FROM rank 
  WHERE rank_max_x = 1 OR rank_min_x = 1 OR rank_max_y = 1 OR rank_min_y = 1;
```
