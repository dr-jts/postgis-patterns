---
parent: Processing
---

# Conflation / Matching
{: .no_toc }

1. TOC
{:toc}


### Adjust polygons to fill a containing Polygon
<https://gis.stackexchange.com/questions/91889/adjusting-polygons-to-boundary-and-filling-holes>

### Match Polygons by Shape Similarity 
https://gis.stackexchange.com/questions/362560/measuring-the-similarity-of-two-polygons-in-postgis

There are different ways to measure the similarity between two polygons such as average distance between the boundaries, Hausdorff distance, Turning Function, Comparing Fourier Transformation of the two polygons

Gives code for Average Boundary Distance

### Find Polygon with more accurate linework
https://gis.stackexchange.com/questions/257052/given-two-polygons-find-the-the-one-with-more-detailed-accurate-shoreline

### Match sets of LineStrings
https://gis.stackexchange.com/questions/347787/compare-two-set-of-linestrings

### Match paths to road network
https://gis.stackexchange.com/questions/349001/aligning-line-with-closest-line-segment-in-postgis

### Match paths
https://gis.stackexchange.com/questions/368146/matching-segments-within-tracks-in-postgis

### Polygon Averaging
https://info.crunchydata.com/blog/polygon-averaging-in-postgis

Solution: Overlay, count “depth” of each resultant, union resultants of desired depth.
