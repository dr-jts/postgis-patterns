# Validation and Cleaning
{: .no_toc }

1. TOC
{:toc}

## Validate Polygons
<https://gis.stackexchange.com/questions/1060/what-are-the-implications-of-invalid-geometries>

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
