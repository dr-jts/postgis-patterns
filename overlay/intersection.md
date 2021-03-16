---
parent: Overlay
---

# Intersection, Clipping, Splitting
{: .no_toc }

1. TOC
{:toc}

## Polygon Intersection

### Aggregated Intersection
<https://gis.stackexchange.com/questions/271824/st-intersection-intersection-of-all-geometries-in-a-table>
<https://gis.stackexchange.com/questions/271941/looping-through-table-to-get-single-intersection-from-n2-geometries-using-postg>
<https://gis.stackexchange.com/questions/60281/query-to-find-the-intersection-coordinates-of-multiple-polyon>
<https://gis.stackexchange.com/questions/269875/aggregate-version-of-st-intersection>

#### Solutions
* Define an aggregate function (given in #2)
* Define a function to do the looping
* Use a recursive CTE (see SQL in #2)

```sql
CREATE AGGREGATE ST_IntersectionAgg (
  basetype = geometry,
  stype = geometry,
  sfunc = ST_Intersection
);
```
**Example:**
```sql
WITH data(geom) AS (VALUES
( ST_Buffer(ST_Point(0,0), 0.75) ),
( ST_Buffer(ST_Point(1,0), 0.75) ),
( ST_Buffer(ST_Point(0.5,1), 0.75) )
)
SELECT ST_IntersectionAgg(geom) FROM data;
```

#### Issues
* How to find all groups of intersecting polygons.  DBSCAN maybe?  (This is suggested in an answer)


### Intersection returning only Polygons
<https://gis.stackexchange.com/questions/89231/postgis-st-intersection-of-polygons-can-return-lines>

Possibly can use `ST_CollectionExtract`?

**See also**
* <https://gis.stackexchange.com/questions/242741/st-intersection-returns-erroneous-polygons> discusses a problem with QGIS visualization caused by the return of a `GEOMETRYCOLLECTION` from `ST_Intersection`.

### Intersection performance - Check containment first
https://postgis.net/2014/03/14/tip_intersection_faster/

### Intersection performance - Use Polygons instead of MultiPolygons
<https://gis.stackexchange.com/questions/101425/using-multipolygon-or-polygon-features-for-large-intersect-operations>

## Clipping

### Clip Districts to Coastline
http://blog.cleverelephant.ca/2019/07/simple-sql-gis.html

### Clip Unioned Polygons to remove water bodies
https://gis.stackexchange.com/questions/331887/unioning-a-set-of-intersections

## Polygon Splitting

https://gis.stackexchange.com/questions/299849/split-polygon-into-separate-polygons-using-a-table-of-individual-lines

### Split rectangles along North-South axis
https://gis.stackexchange.com/questions/239801/how-can-i-split-a-polygon-into-two-equal-parts-along-a-n-s-axis?rq=1

### Split rotated rectangles into equal parts
https://gis.stackexchange.com/questions/286184/splitting-polygon-in-equal-parts-based-on-polygon-area

See also next problem

### Split Polygons into equal-area parts
http://blog.cleverelephant.ca/2018/06/polygon-splitting.html

Does this really result in equal-area subdivision? The Voronoi-of-centroid step is distance-based, not area based…. So may not always work?  Would be good to try this on a bunch of country outines

### Splitting by Line creates narrow gap in Polygonal coverage
https://gis.stackexchange.com/questions/378705/postgis-splitting-polygon-with-line-returns-different-size-polygons-that-creat

Cause is that vertex introduced by splitting is not present in adjacent polygon.

> Perhaps differencing splitting line from surrounding intersecting polygon would introduce that vertex?  
> Or is it better to snap surrounding polygons to split vertices? Probably need a complex process to do this - not really something that can be done easily in DB?

### Splitting Polygon creates invalid coverage with holes
https://gis.stackexchange.com/questions/344716/holes-after-st-split-with-postgis

#### Solution
None so far. Need some way to create a clean coverage
