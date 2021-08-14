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

### Using ROW_NUBER window function
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
