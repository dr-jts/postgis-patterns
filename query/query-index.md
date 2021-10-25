---
parent: Querying
---

# Using Indexes
{: .no_toc }

1. TOC
{:toc}

## Kinds of Spatial Indexes
<https://blog.crunchydata.com/blog/the-many-spatial-indexes-of-postgis>

![](https://blog.crunchydata.com/hs-fs/hubfs/rtree.png?width=543&name=rtree.png)

![](https://blog.crunchydata.com/hs-fs/hubfs/quadtree.png?width=558&name=quadtree.png)

## Miscellaneous

<https://gis.stackexchange.com/questions/172266/improve-performance-of-a-postgis-st-dwithin-query>

<https://gis.stackexchange.com/questions/162651/looking-for-boolean-intersection-of-small-table-with-huge-table>

## Use ST_Intersects instead of ST_DIsjoint
<https://gis.stackexchange.com/questions/167945/delete-polygonal-features-in-one-layer-outside-polygon-feature-of-another-layer>

## Spatial Predicate argument ordering important for indexing
<https://gis.stackexchange.com/questions/209959/how-to-use-st-intersects-with-different-geometry-type>

This may be due to indexing being used for first ST_Intersects arg, but not second?

## Clustering on a spatial Index
<https://gis.stackexchange.com/questions/240721/postgis-performance-increase-with-cluster>

## Use an Index for performance
<https://gis.stackexchange.com/questions/237709/speeding-up-intersect-query-in-postgis>
