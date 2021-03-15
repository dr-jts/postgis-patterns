---
parent: Overlay
---

# Noding

1. TOC
{:toc}

### Node (Split) a table of lines by another table of lines
https://lists.osgeo.org/pipermail/postgis-users/2019-September/043617.html

### Node a table of lines with itself
https://gis.stackexchange.com/questions/368996/intersecting-line-at-junction-in-postgresql

### Node a table of Lines by a Set of Points
https://gis.stackexchange.com/questions/332213/split-lines-with-points-postgis

### Compute points of intersection for a LineString
https://gis.stackexchange.com/questions/16347/is-there-a-postgis-function-for-determining-whether-a-linestring-intersects-itse

### Compute location of noding failures
https://gis.stackexchange.com/questions/345341/get-location-of-postgis-geos-topology-exception

Using ST_Node on set of linestrings produces an error with no indication of where the problem occurs.  Currently ST_Node uses IteratedNoder, which nodes up to 6 times, ahd fails if intersections are still found.  
Solution
Would be possible to report the nodes found in the last pass, which wouild indicate where the problems occur.  

Would be better to eliminate noding errors via snap-rounding, or some other kind of snapping

### Clip Set of LineString by intersection points
https://gis.stackexchange.com/questions/154833/cutting-linestrings-with-points

Uses `ST_Split_Multi` from here: https://github.com/Remi-C/PPPP_utilities/blob/master/postgis/rc_split_multi.sql

### Construct locations where LineStrings self-intersect
https://gis.stackexchange.com/questions/367120/getting-st-issimple-reason-and-detail-similar-to-st-isvaliddetail-in-postgis

Solution
SQL to compute LineString self-intersetions is provided


