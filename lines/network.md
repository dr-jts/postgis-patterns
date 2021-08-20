---
parent: Lines and Networks
---

# Networks
{: .no_toc }

1. TOC
{:toc}

## Find Shortest Path through linear network
<https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi>

### PostGIS Idea
Input Parameters: linear network MultiLineString, start point, end point

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - perhaps: construct Convex Hull, take longest diameter, find shortest path between those points


## Seaway Distances & Routes
<https://www.ausvet.com.au/seaway-distances-with-postgresql>
![](https://www.ausvet.com.au/wp-content/uploads/Blog_images/seaway_1.png)
