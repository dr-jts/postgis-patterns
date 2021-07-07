---
parent: Overlay
---

# Splitting
{: .no_toc }

1. TOC
{:toc}

### Split Polygons by multiple LineStrings
<https://gis.stackexchange.com/questions/299849/split-polygon-into-separate-polygons-using-a-table-of-individual-lines>

![](https://i.stack.imgur.com/yP0rj.png)


### Split rectangles along North-South axis
<https://gis.stackexchange.com/questions/239801/how-can-i-split-a-polygon-into-two-equal-parts-along-a-n-s-axis?rq=1>

### Split rotated rectangles into equal parts
<https://gis.stackexchange.com/questions/286184/splitting-polygon-in-equal-parts-based-on-polygon-area>

See also next problem

### Split Polygons into equal-area parts
<http://blog.cleverelephant.ca/2018/06/polygon-splitting.html>

![](http://blog.cleverelephant.ca/images/2018/poly-split-6.jpg)

Does this really result in equal-area subdivision? The Voronoi-of-centroid step is distance-based, not area based…. So may not always work?  Would be good to try this on a bunch of country outines

### Splitting by Line creates narrow gap in Polygonal coverage
<https://gis.stackexchange.com/questions/378705/postgis-splitting-polygon-with-line-returns-different-size-polygons-that-creat>

The cause is that vertex introduced by splitting is not present in adjacent polygon.

* Perhaps differencing splitting line from surrounding intersecting polygon would introduce that vertex?  
* Or is it better to snap surrounding polygons to split vertices? Probably need a complex process to do this - not really something that can be done easily in DB?

### Splitting Polygon creates invalid coverage with holes
<https://gis.stackexchange.com/questions/344716/holes-after-st-split-with-postgis>

#### Solution
None so far. Need some way to create a clean coverage
