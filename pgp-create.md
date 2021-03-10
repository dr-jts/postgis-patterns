# Create, Edit 
{: .no_toc }

1. TOC
{:toc}

## Geometry Creation

### Use ST_MakePoint or ST_PointFromText
<https://gis.stackexchange.com/questions/122247/st-makepoint-or-st-pointfromtext-to-generate-points>
<https://gis.stackexchange.com/questions/58605/which-function-for-creating-a-point-in-postgis/58630#58630>

**Solution**
ST_MakePoint is much faster

### Collect Lines into a MultiLine in a given order
<https://gis.stackexchange.com/questions/166701/postgis-merging-linestrings-into-multilinestrings-in-a-particular-order>


## Geometry Editing

### Drop Holes from Polygons
https://gis.stackexchange.com/questions/278154/polygons-have-holes-after-pgr-pointsaspolygon

### Drop Holes from MultiPolygons
https://gis.stackexchange.com/questions/348943/simplifying-a-multipolygon-into-one-polygon-respecting-its-outer-boundaries

Solution
https://gis.stackexchange.com/a/349016/14766

Similar
https://gis.stackexchange.com/questions/291374/cut-out-polygons-that-at-least-partially-fall-in-another-polygon

## Cleaning Data

### Validate Polygons
https://gis.stackexchange.com/questions/1060/what-are-the-implications-of-invalid-geometries

### Remove Ring Self-Intersections / MakeValid
https://gis.stackexchange.com/questions/15286/ring-self-intersections-in-postgis
The question and standard answer (buffer(0) are fairly mundane. But note the answer where the user uses MakeValid and keeps only the result polygons with significant area.  Might be a good option to MakeValid?

### Remove Slivers
https://gis.stackexchange.com/questions/289717/fix-slivers-holes-between-polygons-generated-after-removing-spikes

### Remove Slivers after union of non-vector clean Polygons
https://gis.stackexchange.com/questions/351912/st-union-of-set-of-polygons-result-in-polygon-with-very-small-holes

Gives outline for function `cleanPolygon` but does not provide source

### Renove Spikes
https://trac.osgeo.org/postgis/wiki/UsersWikiExamplesSpikeRemover

https://gasparesganga.com/labs/postgis-normalize-geometry/
