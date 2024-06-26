---
parent: Lines and Networks
---

# Networks
{: .no_toc }

1. TOC
{:toc}

## Find Diameter (Longest Path) of Network
<https://gis.stackexchange.com/questions/295199/how-do-i-select-the-longest-connected-lines-from-postgis-st-approximatemedialaxi>

<https://gis.stackexchange.com/questions/409483/simplify-a-branching-line-string/411926>

<https://gis.stackexchange.com/questions/481620/remove-side-branches-from-linestring>

![](https://i.stack.imgur.com/lLqqW.jpg)

### PostGIS Idea
Input Parameters: MultiLineString

Start and End point could be snapped to nearest endpoints if not already in network
Maybe also function to snap a network?

“Longest Shortest Path” - AKA Diameter of a Graph

**Algorithm**
* Pick an endpoint
* Find furthest vertex (longest shortest path over all other vertices)
* Take that vertex, and find longest shortest path to all other vertices)

## Find overshoots in Network
<https://gis.stackexchange.com/questions/433140/detecting-overshoot-lines-in-road-network-flyovers-over-underpasses-in-postgi>

Find overshoots in a set of LineStrings forming a network.  
Equivalent to finding all crossing pairs of lines.

![](https://i.stack.imgur.com/8d2Ab.png)

**Solution**

Use`ST_Crosses` on all intersecting pairs of lines.
Use a [triangle join](https://www.sqlservercentral.com/articles/hidden-rbar-triangular-joins) to ensure checking each pair only once.
Indexes on `id` and geometry` should be used.

```
WITH data(id, geom) AS (VALUES
  (1,ST_GeomFromText('MULTILINESTRING((136.63964778201967 36.58031552554121,136.64078637948796 36.57968013023401,136.6414207216891 36.579313967665655,136.64203092428795 36.578959650067986,136.6428838673968 36.57843301697034,136.6438347098042 36.577844992552514))', 4326)),
  (2,ST_GeomFromText('MULTILINESTRING((136.64039880046448 36.57933335255234,136.64078637948796 36.57968013023401,136.64119407544638 36.580076445271914,136.64165541506554 36.58051798901414))', 4326)),
  (3,ST_GeomFromText('MULTILINESTRING((136.64039880046448 36.57933335255234,136.64044439789086 36.578869185464725))', 4326)),
  (4,ST_GeomFromText('MULTILINESTRING((136.6409003730539 36.57873995108804,136.6414207216891 36.579313967665655,136.64222404380473 36.580108753416425))', 4326)),
  (5,ST_GeomFromText('MULTILINESTRING((136.6416232292288 36.57843409435816,136.64203092428795 36.578959650067986))', 4326)),
  (6,ST_GeomFromText('MULTILINESTRING((136.64203092428795 36.578959650067986,136.64244666738 36.57935489221467,136.64283290461503 36.57974259264671))', 4326)),
  (7,ST_GeomFromText('MULTILINESTRING((136.64250969906357 36.57802376968141,136.6428838673968 36.57843301697034,136.64326608196427 36.578854108330574))', 4326)),
  (8,ST_GeomFromText('MULTILINESTRING((136.64364829653186 36.577513284810436,136.6438347098042 36.577844992552514))', 4326)),
  (9,ST_GeomFromText('MULTILINESTRING((136.6438347098042 36.577844992552514,136.64412438862985 36.57833070559775))', 4326))
)
SELECT a.id, b.id, ST_Intersection(a.geom, b.geom) AS geom 
FROM data a
JOIN data b ON ST_Intersects(a.geom, b.geom)
WHERE a.id < b.id AND ST_Crosses(a.geom, b.geom);
```


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
 
 
