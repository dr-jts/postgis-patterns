---
parent: Lines and Networks
---

# Networks
{: .no_toc }

1. TOC
{:toc}

## Find Shortest Path through linear network
<https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi>

![](https://i.stack.imgur.com/lLqqW.jpg)

### PostGIS Idea
Input Parameters: linear network MultiLineString, start point, end point

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - AKA Diameter of a Graph

**Algorithm**
* Pick an endpoint
* Find furthest vertex (longest shortest path over all other vertices)
* Take that vertex, and find longest shortest path to all other vertices)

## Minimum Spanning Tree of a set of Points
<https://gis.stackexchange.com/questions/418840/creating-polygon-around-set-of-points-without-blank-polygon-spaces-in-between-in>

**Prim's Algorithm**

Recursive query using:
* **vertices** - current set of vertices in tree
* **edge** - the last edge added to the tree
* **ids** - the IDs of the vertices in the tree (allows choosing a vertex not in the tree)

The algorithm works by choosing an unused point as the next vertex to add, 
and connecting it to the tree by the shortest line.

```sql
WITH RECURSIVE
graph AS (
  SELECT  dmp.path[1] AS id, dmp.geom
  FROM ST_Dump('MULTIPOINT ((20 20), (20 60), (40 50), (70 30), (80 70), (50 90), (30 90))'::GEOMETRY) AS dmp
),
tree AS (
  SELECT  g.geom::GEOMETRY AS vertices,
          ARRAY[g.id] AS ids,
          NULL::GEOMETRY AS edge
  FROM graph AS g
  WHERE id = 1
  UNION ALL
  SELECT  ST_Union(t.vertices, v.geom),
          t.ids || v.id,
          ST_ShortestLine(t.vertices, v.geom)
  FROM tree AS t
  CROSS JOIN LATERAL (
    SELECT g.id, g.geom
    FROM graph AS g
    WHERE NOT g.id = ANY(t.ids)
    ORDER BY t.vertices <-> g.geom
    LIMIT 1
  ) AS v
)       
SELECT edge FROM tree WHERE edge IS NOT NULL;
```

## Minimum Spanning Tree

<https://gis.stackexchange.com/questions/374132/finding-the-minimum-spanning-tree-mst>
![](https://i.stack.imgur.com/oqALq.png)

SQL [functions](https://gist.github.com/andrewxhill/13de0618d31893cdc4c5) to compute a Minimum Spanning Tree.

## Seaway Distances & Routes
<https://www.ausvet.com.au/seaway-distances-with-postgresql>
![](https://www.ausvet.com.au/wp-content/uploads/Blog_images/seaway_1.png)

## Find isolated Network edges (1)
<https://gis.stackexchange.com/questions/408528/finding-isolated-links-in-a-network-of-links-with-postgresql>

![](https://i.stack.imgur.com/jx85n.png)

```sql
SELECT l1.id
FROM links l1
WHERE NOT EXISTS
 (SELECT 1 
  FROM links l2
  WHERE l1.id != l2.id
  AND ST_INTERSECTS(l1.geom, l2.geom)
 );
 ```
 ## Find isolated Network edges (2)
 <https://gis.stackexchange.com/questions/408695/compound-contiguous-links-to-lines-in-a-network-and-find-isolated-lines>
 
 ![](https://i.stack.imgur.com/1eoWm.jpg)
 
 
