---
parent: Processing
---

# Point Distributions
{: .no_toc }

1. TOC
{:toc}

### Generate Evenly-Distributed Points in a Polygon

<https://gis.stackexchange.com/questions/8468/creating-evenly-distributed-points-within-an-irregular-boundary>

One solution: create a grid of points and then clip to polygon

See also: 
* <https://math.stackexchange.com/questions/15624/distribute-a-fixed-number-of-points-uniformly-inside-a-polygon>
* <https://gis.stackexchange.com/questions/4663/how-to-create-regular-point-grid-inside-a-polygon-in-postgis>

### Construct Well-spaced Random Points in a Polygon
<https://gis.stackexchange.com/questions/377606/ensuring-all-points-are-a-certain-distance-from-polygon-boundary>

Uses clustering on randomly generated points.  
Suggestion is to use neg-buffered polygon to ensure distance from polygon boundary

A nicer solution will be when PostGIS provides a way to generate random points using [Poisson Disk Sampling](https://www.jasondavies.com/poisson-disc/).

### Place Maximum Number of Points in a Polygon
<https://gis.stackexchange.com/questions/4828/seeking-algorithm-to-place-maximum-number-of-points-within-constrained-area-at-m>

### Sample Linear Point Clouds at given density
<https://gis.stackexchange.com/questions/131854/spacing-a-set-of-points>
