# Shortest path search using PROLOG

This note shows how to find shortest paths over the RSM topology port graph using PROLOG, as a complement to OWL reasoners and SPARQL engines (see `path_queries.md` for the SPARQL counterpart). No benchmarking is attempted at this stage.

A runnable version of the demo below is available online on SWISH (SWI-Prolog for SHaring): [https://swish.swi-prolog.org/p/240606%20Navigabilities.pl](https://swish.swi-prolog.org/p/240606%20Navigabilities.pl).

## PROLOG in a nutshell

For readers unfamiliar with PROLOG, the few constructs used here:

- `p(a)` is a fact (can be interpreted as unary property "thing a has property p"; similarly p(a, b) would represent a binary property, etc.);`p(X)` with capitalized `X` is a goal with a variable. Constants start with lower case, variables with upper case.
- `A :- B` reads "A holds if B holds" (a rule).
- `,` is logical AND; `;` is logical OR.
- `[H|T]` denotes a list with head `H` and tail `T`.
- PROLOG answers queries by trying to *prove* goals, backtracking through all alternatives. Asking for all proofs of `find_path(a1, d1, Path)` therefore enumerates all paths.
- `\+ A` means "A cannot be proven", which PROLOG treats as falsity ("negation as failure") — a frequent source of misinterpretation.
- The `:- table p/2 .` directive memoizes predicate `p` and, importantly, prevents infinite loops when rules are recursive (e.g. transitivity over a cyclic graph).

## From the ontology to PROLOG facts

The topology ontology maps naturally onto PROLOG predicates:


| Ontology                              | PROLOG                                                | Remark                 |
| ------------------------------------- | ----------------------------------------------------- | ---------------------- |
| `topo:LinearElement` individual       | `linearelement/1` fact                                |                        |
| `topo:port` (port *p* on element *x*) | `port(P, X)` fact                                     |                        |
| `topo:connectedWith`                  | `connectedWith/2` facts + symmetry/transitivity rules |                        |
| `topo:navigableTo`                    | `navigableTo/2` facts                                 | directed edges         |
| `topo:navigableToTransitive`          | `navigableToTransitive/2` rules                       | inferred, not asserted |
| `rgeo:hasMetricLength`                | `elementlength/2` facts                               | see below              |


Lengths: every `topo:LinearElement` is also a `rgeo:Feature`, and we assume each has a usable `rgeo:hasMetricLength` value. In PROLOG, each such value becomes a fact `elementlength(Element, Length)`. (In a production setting, these facts would be extracted from the triple store by a SPARQL `SELECT` and asserted into the PROLOG knowledge base.)

## Example network

Four linear elements `a`, `b`, `c`, `d`: `a` on the left, `d` on the right, `b` and `c` in parallel in the middle (station tracks: `c` the "through" track, `b` the siding), joined by switches at both ends. A loop `e` is attached at the right end of `d`. Switches are not topology elements; their presence materializes as navigabilities.

Each linear element `x` has two ports, conventionally named `x0` and `x1`.

```prolog
linearelement(a) .
linearelement(b) .
linearelement(c) .
linearelement(d) .
linearelement(e) .   % the loop

port(a0, a) .  port(a1, a) .
port(b0, b) .  port(b1, b) .
port(c0, c) .  port(c1, c) .
port(d0, d) .  port(d1, d) .
port(e0, e) .  port(e1, e) .   % a loop's two ports coincide geometrically, but remain distinct
```

Connections (the topology layout; symmetric and transitive):

```prolog
:- table connectedWith/2 .   % prevents infinite recursion

connectedWith(a1, b0) .
connectedWith(a1, c0) .
connectedWith(b1, d0) .
connectedWith(c1, d0) .
connectedWith(d1, e0) .
connectedWith(d1, e1) .      % together with the previous line: a loop

connectedWith(X, Y) :- connectedWith(Y, X).                       % symmetry
connectedWith(X, Y) :- connectedWith(X, Z), connectedWith(Z, Y).  % transitivity
```

Navigabilities (directed; expressed "exit port to exit port", which memorizes the travel direction and makes transitivity meaningful):

```prolog
:- table navigableToTransitive/2 .

navigableToTransitive(X, Y) :- navigableTo(X, Y).
navigableToTransitive(X, Y) :- navigableToTransitive(X, Z), navigableTo(Z, Y).

navigableTo(a1, b1) .   navigableTo(b0, a0) .
navigableTo(a1, c1) .   navigableTo(c0, a0) .
navigableTo(b1, d1) .   navigableTo(d0, b0) .
navigableTo(c1, d1) .   navigableTo(d0, c0) .

% Loop e: entry only at e0 side, exit to e1 (e.g. a sprung switch):
navigableTo(d1, e1) .
navigableTo(e1, d0) .
```

Note the distinction between asserted `navigableTo` facts and inferred `navigableToTransitive`: the same distinction the ontology makes, and good practice for data management (compare SKOS).

Element lengths, i.e. the `rgeo:hasMetricLength` values:

```prolog
elementlength(a, 100) .
elementlength(b, 220) .   % the siding is a bit longer...
elementlength(c, 200) .   % ...than the straight through track
elementlength(d, 150) .
elementlength(e, 450) .
```

## Path search and shortest path

A path is a list of ports. Path enumeration is two rules — direct navigability gives a two-port path, otherwise take one navigable hop and recurse:

```prolog
find_path(Start, End, [Start, End]) :-
    navigableTo(Start, End).
find_path(Start, End, [Start|Rest]) :-
    navigableTo(Start, Next),
    Start \= End,
    find_path(Next, End, Rest).
```

Path length sums the lengths of the elements whose ports appear in the path:

```prolog
elementlength_of_port(Port, Length) :-
    port(Port, Element),
    elementlength(Element, Length).

path_length([], 0).
path_length([H|T], TotalLength) :-
    elementlength_of_port(H, Length),
    path_length(T, RestLength),
    TotalLength is Length + RestLength.
```

All paths, then the shortest one ("without Edsger Dijkstra's help"):

```prolog
find_all_paths(Start, End, Paths) :-
    findall(Path, find_path(Start, End, Path), Paths).

find_shortest_path(Start, End, ShortestPath, MinLength) :-
    find_all_paths(Start, End, Paths),
    maplist(path_length, Paths, Lengths),
    min_member(MinLength, Lengths),
    nth0(Index, Lengths, MinLength),
    nth0(Index, Paths, ShortestPath).
```

### Sample queries

```prolog
% Does a route exist from a1 to d1? (existence, like navigableToTransitive)
?- navigableToTransitive(a1, d1).

% Enumerate all paths from a1 to d1 (composition):
?- find_all_paths(a1, d1, Paths).

% Shortest path and its length:
?- find_shortest_path(a1, d1, Path, Length).
% Path = [a1, c1, d1], Length = 450
% (the through track c wins over the siding b)
```

Note: `maplist`, `sum_list`, `min_member`, `nth0` are SWI-Prolog library predicates, not necessarily ISO PROLOG.

## Inference-based path search vs. Dijkstra encoded in PROLOG

The approach above and a Dijkstra implementation both run inside a PROLOG engine, but they are fundamentally different in nature.

### What the inference-based approach does

`find_path/3` is a *declarative specification* of what a path is; the PROLOG engine's resolution and backtracking do the searching. The strategy amounts to **exhaustive depth-first enumeration**:

- `find_all_paths/3` materializes *every* acyclic route, then `find_shortest_path/4` compares their lengths.
- Complexity is driven by the number of distinct paths, which can grow **exponentially** with network size (each station with parallel tracks doubles the count downstream).
- Correctness relies on the search terminating: in the plain (untabled) recursive rule, cycles like the loop `e` only terminate because the navigability orientation prevents revisiting; in less disciplined graphs, tabling (`:- table`) or an explicit visited-list is needed.
- Its strengths are elsewhere: the code is a near-verbatim transcription of the ontology's semantics (`navigableTo`, transitivity), it answers *existence* queries (`navigableToTransitive/2`) very cheaply once tabled, and it returns *all* solutions for free via backtracking — useful for validating the model, not just routing through it.

### What a Dijkstra encoding does

Dijkstra's algorithm is an *imperative procedure* (greedy best-first search with a priority queue) that one can transcribe into PROLOG:

- It maintains a frontier of partial paths ordered by accumulated length and always expands the cheapest one, settling each node at most once.
- Complexity is **polynomial** — O((V+E) log V) with a proper priority queue — regardless of how many distinct paths exist.
- It finds *one* shortest path (or the shortest-path tree from a source); it does not naturally enumerate all paths and does not handle negative weights (irrelevant for track lengths).
- The PROLOG code is longer and less declarative: the priority queue, the settled set, and the relaxation step are bookkeeping, not knowledge representation. The elegance of "rules as documentation" is largely lost — the same algorithm could equally be written in any language, with PROLOG providing little leverage.

### Summary


| Aspect                      | Inference (`find_path` + min)                           | Dijkstra in PROLOG                  |
| --------------------------- | ------------------------------------------------------- | ----------------------------------- |
| Style                       | Declarative; mirrors ontology semantics                 | Procedural; algorithm transcription |
| Strategy                    | Exhaustive enumeration + comparison                     | Greedy best-first, settle-once      |
| Complexity                  | Exponential in path count                               | Polynomial (O((V+E) log V))         |
| Output                      | All paths; existence checks; shortest by post-selection | One shortest path (or tree)         |
| Cycle handling              | Tabling / orientation discipline                        | Settled set handles cycles natively |
| Code as model documentation | Yes — rules read like the ontology                      | No — mostly bookkeeping             |
| Suitable scale              | Small/medium networks, model validation                 | Large networks, operational routing |


In short: the inference-based approach is the right tool for **checking the model** — does navigability behave as intended, which routes exist, are non-navigabilities respected — on networks small enough to enumerate. Dijkstra (or A*, or bidirectional variants) is the right tool when shortest-path computation itself is the product, on networks where exhaustive enumeration is intractable. Both can coexist: validate with rules, route with the algorithm.

No benchmarking has been carried out at this stage; the complexity arguments above are standard results, not measurements.