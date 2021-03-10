# Transform
{: .no_toc }

1. TOC
{:toc}

## Transformation

### Scale polygon around a given point
https://gis.stackexchange.com/questions/227435/postgis-scaling-for-polygons-at-a-fixed-center-location

No solution so far
Issues
SQL given is overly complex and inefficient.  But idea is right

## Simplification
https://gis.stackexchange.com/questions/293429/decrease-polygon-vertices-count-maintaining-its-aspect

## Smoothing
https://gis.stackexchange.com/questions/313667/postgis-snap-line-segment-endpoint-to-closest-other-line-segment

Problem is to smooth a network of lines.  Network is not fully noded, so smoothing causes touching lines to become disconnected.
#### Solution
Probably to node the network before smoothing.
Not sure how to node the network and preserve IDs however!?

## Coordinate Systems

### Find a good planar projection
https://gis.stackexchange.com/questions/275057/how-to-find-good-meter-based-projection-in-postgis

Also https://gis.stackexchange.com/questions/341243/postgis-buffer-in-meters-without-geography

### ST_Transform creates invalid geometry
https://gis.stackexchange.com/questions/341160/why-do-two-tables-with-valid-geometry-return-self-intersection-errors-when-inter

Also:  https://trac.osgeo.org/postgis/ticket/4755
Has an example geometry which becomes invalid under transform

