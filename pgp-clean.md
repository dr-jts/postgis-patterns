# Validation and Cleaning
{: .no_toc }

1. TOC
{:toc}

## Constraint to Validate Geometry column
<https://gis.stackexchange.com/questions/1060/what-are-the-implications-of-invalid-geometries>
<https://gis.stackexchange.com/a/11234/14766>

```sql
ALTER TABLE public.my_valid_table
  ADD CONSTRAINT enforce_valid_geom CHECK (st_isvalid(geom));
```

## Remove Ring Self-Intersections / MakeValid
<https://gis.stackexchange.com/questions/15286/ring-self-intersections-in-postgis>
The question and standard answer (buffer(0) are fairly mundane. But note the answer where the user uses MakeValid and keeps only the result polygons with significant area.  Might be a good option to MakeValid?

## Remove Slivers
<https://gis.stackexchange.com/questions/289717/fix-slivers-holes-between-polygons-generated-after-removing-spikes>

## Remove Slivers after union of non-vector clean Polygons
<https://gis.stackexchange.com/questions/351912/st-union-of-set-of-polygons-result-in-polygon-with-very-small-holes>

Gives outline for function `cleanPolygon` but does not provide source

## Remove Spikes
<https://gis.stackexchange.com/questions/173977/how-to-remove-spikes-in-polygons-with-postgis>

![](https://i.stack.imgur.com/iCRwL.png)

**Solutions**

Some custom code implementations:

<https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesSpikeRemover>

<https://gasparesganga.com/labs/postgis-normalize-geometry/>

## Remove vertices along straight edges
<https://gis.stackexchange.com/questions/15127/how-to-rectify-the-walls-of-buildings-in-postgis>

![](https://i.stack.imgur.com/GQB3b.jpg)

**Solution**

* Simplify the polygon with a very small distance tolerance, using one of: 
  * [`ST_Simplify`](https://postgis.net/docs/manual-3.3/ST_Simplify.html)
  * [`ST_SimplifyPreserveTopology`](https://postgis.net/docs/manual-3.3/ST_SimplifyPreserveTopology.html) 
  * [`ST_SimplifyVW`](https://postgis.net/docs/manual-3.3/ST_SimplifyVW.html) 
  * [`ST_SimplifyPolygonHull`](https://postgis.net/docs/manual-3.3/ST_SimplifyPolygonHull.html) 
