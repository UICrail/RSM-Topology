# Path queries over ports and navigability

This note describes SPARQL patterns for answering two questions over the RSM Topology port graph:

1. **Route existence** — is there a directed route from a start port to an end port?
2. **Route composition** — what are the routes (sequences of ports) between them?

In the topology ontology, ports are `topo:Port` individuals. Directed edges are expressed primarily by `topo:navigableTo` (direct navigability from one port to another). The ontology also declares `topo:navigableToTransitive` as the OWL transitive closure of `topo:navigableTo`.

Throughout this document, replace the placeholder IRIs with your own port identifiers:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

# Example endpoints
# <urn:port:A>  start port
# <urn:port:D>  end port
```

## Choosing `navigableTo` vs `navigableToTransitive`


| Approach                                     | What it uses                                                     | When to use                                                                                                       |
| -------------------------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `topo:navigableTo` (and `+` / `*` in SPARQL) | Asserted direct edges; transitivity computed by the query engine | Default for **route composition** (you need intermediate ports) and for **existence** on asserted data            |
| `topo:navigableToTransitive`                 | Inferred closure triples (materialized by a reasoner)            | **Existence only**, when the repository has reasoning enabled and you only need reachability, not the actual hops |


**Important distinctions:**

- `navigableTo+` at query time and `navigableToTransitive` answer the same reachability question on a fully reasoned graph, but only `navigableTo` (or store-specific path enumeration) yields **intermediate ports**.
- `navigableToTransitive` triples are typically **not** stored unless inference is run (e.g. HermiT in CI produces `reports/reasoned.owl`). Querying `navigableToTransitive` against asserted-only data may return no results even when a multi-hop route exists.
- If edges are reified as `topo:Navigability` individuals (`topo:fromPort` / `topo:toPort`) rather than direct `navigableTo` triples, extend the graph patterns in the examples below with a `UNION` branch (see [Reified navigability](#reified-navigability)).

---

## Purpose

### Route existence (boolean)

Determine whether a directed path exists from port *A* to port *D* using `navigableTo` as the edge relation.

- **With `navigableTo`:** existence is equivalent to `A topo:navigableTo+ D` (one or more hops).
- **With `navigableToTransitive`:** existence is equivalent to `A topo:navigableToTransitive D` (single triple, requires inferred data).

Use `*` instead of `+` if the start and end port may be the same node (zero-hop path).

### Route composition (list of paths)

Return one or more routes, each a sequence of ports `[A, …, D]`, following only `navigableTo` edges.

- Standard SPARQL 1.1 property paths can prove reachability but **do not enumerate** intermediate nodes or all distinct paths.
- GraphDB (graph path search) and Virtuoso (`OPTION (TRANSITIVE, …)`) can return per-hop bindings and path identifiers.
- `navigableToTransitive` alone is **not sufficient** for composition: it collapses all routes into one reachability fact and carries no hop structure.

---

## Using SPARQL 1.1

Portable SPARQL 1.1 works on any compliant endpoint. It is the baseline for **existence**; for **all routes** it is limited to fixed maximum depth unless combined with engine-specific extensions (see following sections).

### Route existence with `navigableTo`

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableTo+ <urn:port:D> .
}
```

Or as a `SELECT` boolean:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT (EXISTS { <urn:port:A> topo:navigableTo+ <urn:port:D> } AS ?routeExists) {}
```

### Route existence with `navigableToTransitive`

Requires `topo:navigableToTransitive` triples in the dataset (reasoning materialized):

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableToTransitive <urn:port:D> .
}
```

Or:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT (EXISTS {
  <urn:port:A> topo:navigableToTransitive <urn:port:D>
} AS ?routeExists) {}
```

On asserted-only data, use `navigableTo+` instead.

### Route composition with `navigableTo` (fixed depth)

Enumerate paths up to a chosen maximum length by unrolling hops with `UNION`:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathLength ?route WHERE {
  BIND(<urn:port:A> AS ?startPort)
  BIND(<urn:port:D> AS ?endPort)

  {
    # length 1
    BIND(1 AS ?pathLength)
    ?startPort topo:navigableTo ?endPort .
    BIND(CONCAT(STR(?startPort), " -> ", STR(?endPort)) AS ?route)
  }
  UNION {
    # length 2
    BIND(2 AS ?pathLength)
    ?startPort topo:navigableTo ?p1 .
    ?p1 topo:navigableTo ?endPort .
    BIND(CONCAT(STR(?startPort), " -> ", STR(?p1), " -> ", STR(?endPort)) AS ?route)
  }
  UNION {
    # length 3
    BIND(3 AS ?pathLength)
    ?startPort topo:navigableTo ?p1 .
    ?p1 topo:navigableTo ?p2 .
    ?p2 topo:navigableTo ?endPort .
    BIND(CONCAT(STR(?startPort), " -> ", STR(?p1), " -> ", STR(?p2), " -> ", STR(?endPort)) AS ?route)
  }
}
ORDER BY ?pathLength
```

Add further `UNION` branches for longer paths. This pattern is verbose but portable and returns explicit port sequences.

### Route composition with `navigableToTransitive`

**Not applicable for listing routes.** A `navigableToTransitive` triple only states that *some* route exists; it does not identify hops or distinguish multiple routes. Use `navigableTo` (fixed depth or engine-specific enumeration) for composition.

For existence-only checks, `navigableToTransitive` is equivalent to `navigableTo+` when the transitive closure is fully materialized.

### SPARQL 1.1 limitations

- Property paths (`+`, `*`) do not bind intermediate variables or return all simple paths.
- No standard recursive path enumeration in SPARQL 1.1.
- Branching topologies can yield multiple routes; fixed-depth `UNION` must be extended manually for each hop count.

---

## Using GraphDB

Tested against **GraphDB 11.2**. Graph path search is documented in the [GraphDB graph path search guide](https://graphdb.ontotext.com/documentation/11.2/graph-path-search.html).

**Prerequisite:** enable the **Graph path search** plugin for the repository.

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>
```

### Route existence with `navigableTo`

Standard `ASK` (no plugin required):

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableTo+ <urn:port:D> .
}
```

GraphDB-native shortest-distance check (graph pattern restricts traversal to `navigableTo`):

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>

ASK {
  SERVICE <http://www.ontotext.com/path#search> {
    <urn:path> path:findPath path:distance ;
               path:sourceNode      <urn:port:A> ;
               path:destinationNode <urn:port:D> ;
               path:distanceBinding ?dist .
    SERVICE <urn:path> {
      ?s topo:navigableTo ?o .
    }
  }
}
```

When the query succeeds, `?dist` is the number of hops on a shortest route.

### Route existence with `navigableToTransitive`

When inference is enabled and `navigableToTransitive` triples are present:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableToTransitive <urn:port:D> .
}
```

GraphDB path search can also traverse inferred edges if the repository is configured to include them in path evaluation. Restrict the inner pattern to the transitive property:

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>

ASK {
  SERVICE <http://www.ontotext.com/path#search> {
    <urn:path> path:findPath path:distance ;
               path:sourceNode      <urn:port:A> ;
               path:destinationNode <urn:port:D> ;
               path:distanceBinding ?dist .
    SERVICE <urn:path> {
      ?s topo:navigableToTransitive ?o .
      FILTER(?s != ?o)   # exclude zero-length if undesired
    }
  }
}
```

Note: with `navigableToTransitive` as the only edge predicate, path search returns a **hop count along transitive arcs**, not the fine-grained sequence of direct `navigableTo` steps. For true route composition, use `navigableTo` in the patterns below.

### Route composition with `navigableTo`

**Per-edge bindings** (recommended for reconstructing port sequences):

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathIndex ?edgeIndex ?from ?to
WHERE {
  BIND(<urn:port:A> AS ?startPort)
  BIND(<urn:port:D> AS ?endPort)

  SERVICE <http://www.ontotext.com/path#search> {
    <urn:path> path:findPath path:allPaths ;
               path:sourceNode         ?startPort ;
               path:destinationNode    ?endPort ;
               path:pathIndex          ?pathIndex ;
               path:startNode          ?from ;
               path:endNode            ?to ;
               path:resultBindingIndex ?edgeIndex ;
               path:maxPathLength      20 .

    SERVICE <urn:path> {
      ?from topo:navigableTo ?to .
    }
  }
}
ORDER BY ?pathIndex ?edgeIndex
```


| Variable        | Meaning                       |
| --------------- | ----------------------------- |
| `?pathIndex`    | Route identifier (0, 1, 2, …) |
| `?edgeIndex`    | Hop index within the route    |
| `?from` / `?to` | Ports on each hop             |


Reconstruct each route: take `?from` where `?edgeIndex = 0`, then append each `?to` ordered by `?edgeIndex`.

**Aggregated string per route:**

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathIndex (COUNT(?edgeIndex) AS ?hops)
       (CONCAT(STR(?startPort), " -> ", GROUP_CONCAT(STR(?to); separator=" -> ")) AS ?route)
WHERE {
  BIND(<urn:port:A> AS ?startPort)
  BIND(<urn:port:D> AS ?endPort)

  SERVICE <http://www.ontotext.com/path#search> {
    <urn:path> path:findPath path:allPaths ;
               path:sourceNode         ?startPort ;
               path:destinationNode    ?endPort ;
               path:pathIndex          ?pathIndex ;
               path:startNode          ?from ;
               path:endNode            ?to ;
               path:resultBindingIndex ?edgeIndex ;
               path:maxPathLength      20 .
    SERVICE <urn:path> {
      ?from topo:navigableTo ?to .
    }
  }
}
GROUP BY ?pathIndex ?startPort
ORDER BY ?pathIndex
```

### Route composition with `navigableToTransitive`

**Not recommended for listing routes** for the same reason as in SPARQL 1.1: transitive arcs do not decompose into direct `navigableTo` hops. If the graph contains only (or primarily) `navigableToTransitive` triples and you need port sequences, either:

- query over asserted `navigableTo` and let path search walk direct edges, or
- run reasoning that keeps direct `navigableTo` triples alongside inferred `navigableToTransitive`.

If you only need to confirm that a chain of transitive links exists (not the underlying direct steps), you can enumerate `navigableToTransitive` arcs:

```sparql
PREFIX path: <http://www.ontotext.com/path#>
PREFIX topo:  <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathIndex ?edgeIndex ?from ?to
WHERE {
  BIND(<urn:port:A> AS ?startPort)
  BIND(<urn:port:D> AS ?endPort)

  SERVICE <http://www.ontotext.com/path#search> {
    <urn:path> path:findPath path:allPaths ;
               path:sourceNode         ?startPort ;
               path:destinationNode    ?endPort ;
               path:pathIndex          ?pathIndex ;
               path:startNode          ?from ;
               path:endNode            ?to ;
               path:resultBindingIndex ?edgeIndex ;
               path:maxPathLength      20 .
    SERVICE <urn:path> {
      ?from topo:navigableToTransitive ?to .
      FILTER(?from != ?to)
    }
  }
}
ORDER BY ?pathIndex ?edgeIndex
```

Treat results as **transitive-step** sequences, not guaranteed decompositions into direct navigability links.

### GraphDB notes

- Default `path:maxPathLength` is **8**; set a higher value or `-1` for unlimited (potentially expensive).
- Use a nested graph pattern (`?from topo:navigableTo ?to`) rather than a wildcard predicate, so traversal follows only navigability edges.
- Alternative service IRI: `SERVICE path:search { ... }` with `PREFIX path: <http://www.ontotext.com/path#>`.

---

## Using Virtuoso

Virtuoso provides transitivity through `OPTION (TRANSITIVE, …)` on a subquery. See the [Virtuoso transitivity documentation](http://docs.openlinksw.com/virtuoso/rdfsparqlimplementatiotrans/).

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>
```

### Route existence with `navigableTo`

Property path:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableTo+ <urn:port:D> .
}
```

Transitivity option:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  {
    SELECT ?from ?to
    WHERE {
      ?from topo:navigableTo ?to .
    }
  }
  OPTION (
    TRANSITIVE,
    T_IN  (?from),
    T_OUT (?to),
    T_MIN (1)
  )
  FILTER (?from = <urn:port:A> && ?to = <urn:port:D>)
}
```

Shorthand on a single triple pattern:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT ?to
WHERE {
  <urn:port:A> topo:navigableTo ?to
    OPTION (TRANSITIVE, T_MIN(1), T_MAX(50)) .
  FILTER (?to = <urn:port:D>)
}
```

### Route existence with `navigableToTransitive`

When transitive triples are materialized:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  <urn:port:A> topo:navigableToTransitive <urn:port:D> .
}
```

Or via transitivity over the transitive property itself (one step if closure is complete):

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

ASK {
  {
    SELECT ?from ?to
    WHERE {
      ?from topo:navigableToTransitive ?to .
      FILTER(?from != ?to)
    }
  }
  OPTION (
    TRANSITIVE,
    T_IN  (?from),
    T_OUT (?to),
    T_MIN (1)
  )
  FILTER (?from = <urn:port:A> && ?to = <urn:port:D>)
}
```

On asserted-only data, prefer `navigableTo` with `OPTION (TRANSITIVE, …)` or `navigableTo+`.

### Route composition with `navigableTo`

Use `OPTION (TRANSITIVE, …)` **without** `T_SHORTEST_ONLY`. Use `path_id` and `step_no` to distinguish routes and hop order. `T_NO_CYCLES` avoids revisiting ports.

**Per-port bindings:**

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathId ?step ?port
WHERE {
  {
    SELECT ?from ?to ?port
    WHERE {
      ?from topo:navigableTo ?to .
      BIND(?from AS ?port)
    }
  }
  OPTION (
    TRANSITIVE,
    T_IN  (?from),
    T_OUT (?to),
    T_NO_CYCLES,
    T_MIN (1),
    T_MAX (50),
    T_STEP (?port)     AS ?port,
    T_STEP ('path_id') AS ?pathId,
    T_STEP ('step_no') AS ?step
  )
  FILTER (?from = <urn:port:A> && ?to = <urn:port:D>)
}
ORDER BY ?pathId ?step
```

**Aggregated sequence per route** (append the destination port):

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathId
       (GROUP_CONCAT(STR(?port); separator=" -> ") AS ?route)
WHERE {
  {
    SELECT ?from ?to ?port ?pathId ?step
    WHERE {
      ?from topo:navigableTo ?to .
      BIND(?from AS ?port)
    }
    OPTION (
      TRANSITIVE,
      T_IN (?from), T_OUT (?to),
      T_NO_CYCLES, T_MIN(1), T_MAX(50),
      T_STEP (?port)     AS ?port,
      T_STEP ('path_id') AS ?pathId,
      T_STEP ('step_no') AS ?step
    )
    FILTER (?from = <urn:port:A>)
  }

  UNION

  {
    SELECT ?to AS ?port ?pathId (?step + 1 AS ?step)
    WHERE {
      ?from topo:navigableTo ?to .
    }
    OPTION (
      TRANSITIVE,
      T_IN (?from), T_OUT (?to),
      T_NO_CYCLES, T_MIN(1), T_MAX(50),
      T_STEP ('path_id') AS ?pathId,
      T_STEP ('step_no') AS ?step
    )
    FILTER (?from = <urn:port:A> && ?to = <urn:port:D>)
  }
}
GROUP BY ?pathId
ORDER BY ?pathId
```

### Route composition with `navigableToTransitive`

As with GraphDB and SPARQL 1.1, `**navigableToTransitive` does not yield direct-hop routes**. Use `navigableTo` in the transitivity subquery for true composition.

If only transitive arcs are available and you need *some* step sequence (not necessarily direct links), substitute `navigableToTransitive` in the subquery and add `FILTER(?from != ?to)`:

```sparql
PREFIX topo: <https://cdm.ovh/rsm/topology/topo#>

SELECT ?pathId ?step ?port
WHERE {
  {
    SELECT ?from ?to ?port
    WHERE {
      ?from topo:navigableToTransitive ?to .
      FILTER(?from != ?to)
      BIND(?from AS ?port)
    }
  }
  OPTION (
    TRANSITIVE,
    T_IN  (?from),
    T_OUT (?to),
    T_NO_CYCLES,
    T_MIN (1),
    T_MAX (50),
    T_STEP (?port)     AS ?port,
    T_STEP ('path_id') AS ?pathId,
    T_STEP ('step_no') AS ?step
  )
  FILTER (?from = <urn:port:A> && ?to = <urn:port:D>)
}
ORDER BY ?pathId ?step
```

### Virtuoso notes

- Do **not** use `T_SHORTEST_ONLY` when enumerating all routes.
- `T_MAX` limits depth; essential on large graphs.
- `T_DISTINCT` limits node revisits and can prune search; usually omit for full path enumeration.
- Virtuoso transitivity requires a **bound start** (`?from`); it is not an unbound all-pairs search.

---

## Reified navigability

If some edges are expressed only as `topo:Navigability` individuals, extend inner graph patterns with:

```sparql
{
  ?from topo:navigableTo ?to .
}
UNION
{
  ?nav a topo:Navigability ;
       topo:fromPort ?from ;
       topo:toPort   ?to .
}
```

This applies to SPARQL 1.1 fixed-depth queries and to GraphDB nested `SERVICE <urn:path>` patterns. It does not produce `navigableToTransitive` triples; reasoning over the ontology may still infer transitive closure separately.

---

## Summary


| Goal                                 | `navigableTo`                                  | `navigableToTransitive`                                  |
| ------------------------------------ | ---------------------------------------------- | -------------------------------------------------------- |
| Exists? (SPARQL 1.1)                 | `ASK { A topo:navigableTo+ D }`                | `ASK { A topo:navigableToTransitive D }` (inferred data) |
| All direct-hop routes (SPARQL 1.1)   | Fixed-depth `UNION`                            | Not applicable                                           |
| Exists? (GraphDB 11.2)               | `ASK` or `path:distance` over `navigableTo`    | `ASK` or `path:distance` over `navigableToTransitive`    |
| All direct-hop routes (GraphDB 11.2) | `path:allPaths` + `?from topo:navigableTo ?to` | Not applicable for direct hops                           |
| Exists? (Virtuoso)                   | `navigableTo+` or `OPTION (TRANSITIVE, …)`     | `ASK` on materialized transitive triples                 |
| All direct-hop routes (Virtuoso)     | `OPTION (TRANSITIVE, …, T_STEP('path_id'), …)` | Not applicable for direct hops                           |


