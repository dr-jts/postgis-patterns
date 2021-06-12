---
parent: Processing
---

# Buffers
{: .no_toc }

1. TOC
{:toc}

### Variable Width Buffer
<https://gis.stackexchange.com/questions/340968/varying-size-buffer-along-a-line-with-postgis>

![](https://i.stack.imgur.com/sUKMj.png)

### Expand a rectangular polygon
<https://gis.stackexchange.com/questions/308333/expanding-polygon-by-distance-using-postgis>

Use `join=mitre mitre_limit=<buffer distance>`

```sql
SELECT ST_Buffer(
        'LINESTRING(5 5,15 15,15 5)'
           ), 1, 'endcap=square join=mitre mitre_limit=1');
```

### Buffer Coastlines with inlet skeletons
<https://gis.stackexchange.com/questions/300867/how-can-i-buffer-a-mulipolygon-only-on-the-coastline>

### Remove Line Buffer artifacts
<https://gis.stackexchange.com/questions/363025/how-to-run-a-moving-window-function-in-a-conditional-statement-in-postgis-for-bu>

Quite bizarre, but apparently works.
