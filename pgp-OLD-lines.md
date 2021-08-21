---
nav_exclude: true
---

# Lines and Networks
{: .no_toc }

1. TOC
{:toc}

## Linear Referencing/Line Handling

### Select every Nth point from a LineString
https://stackoverflow.com/questions/60319473/postgis-how-do-i-select-every-second-point-from-linestring

### Extrapolate Lines
https://gis.stackexchange.com/questions/33055/extrapolating-a-line-in-postgis

ST_LineInterpolatePoint should be enhanced to allow fractions outside [0,1].

### Extend a LineString to the boundary of a polygon
https://gis.stackexchange.com/questions/345463/how-can-i-extend-a-linestring-to-the-edge-of-an-enclosing-polygon-in-postgis

Ideas
A function ST_LineExtract(line, index1, index2) to extract a portion of a LineString between two indices





### Extract Line Segments
<https://gis.stackexchange.com/questions/174472/in-postgis-how-to-split-linestrings-into-their-individual-segments>

#### PostGIS Idea
Create an `ST_DumpSegments` to do this.






## Networks
### Find Shortest Path through linear network
Input Parameters: linear network MultiLineString, start point, end point

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - perhaps: construct Convex Hull, take longest diameter, find shortest path between those points

https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi

## Routing / Pathfinding

### Seaway Distances & Routes
https://www.ausvet.com.au/seaway-distances-with-postgresql/
![](https://www.ausvet.com.au/wp-content/uploads/Blog_images/seaway_1.png)

## Temporal Trajectories

### Find Coincident Paths
https://www.cybertec-postgresql.com/en/intersecting-gps-tracks-to-identify-infected-individuals/

### Remove Stationary Points
https://gis.stackexchange.com/questions/290243/remove-points-where-user-was-stationary

