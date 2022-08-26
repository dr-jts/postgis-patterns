# Anti-Patterns
{: .no_toc }

1. TOC
{:toc}

## Using `ST_Intersects/ST_Buffer` for proximity queries

Use `ST_DWithin` instead, since it is automatically indexed and is much faster.
