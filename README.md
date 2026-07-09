# Purpose

Ontology for the railway network topology. This is part of sRSM (semantic Rail System Model), the successor of RTM/RSM (IRS30100).

# Main features

## Fundamental feature

sRSM basically represents the network as a "line graph": the graph nodes are linear elements; the graph edges are immaterial and express the connections between linear elements. Every physical part of the network is a graph node. This representation can be opposed to the seemingly more intuitive representation (see e.g. OpenStreetMap) where nodes are junctions (switches, crossings) or end points (buffers...) and edges are track segments joining the nodes.

The line graph paradigm of RTM/RSM (IRS30110) met some success, as it was subsequently adopted by railML3 (forking RTM), the RINF Ontology of the European Railway Agency, etc.

> Note: switches and crossings are not part of the RTM/RSM or sRSM topology. They are technical devices allowing rolling stock to actually move from a linear element to the next. Switches can be described separately (see Track Functional ontology) and can be mapped to parts of linear elements; for instance, a switch panel (the physical instance) may extend over parts of three different linear elements (an incoming track on toe side, and on heel side, a through track and a diverted track).

## Levels of detail

RSM-Topology allows the description of the network topology at various levels of details:

* track level: stretches of tracks starting or ending at switches and crossings, or at buffers, or simply nowhere. This level of detail corresponds to the former "MICRO" level of RTM.
* any aggregated level, with a corresponding loss of detail. The aggregation is chosen by the user; it may correspond to the "MESO" level of RTM/RSM (stations or yards connected by tracks) or "MACRO" (stations or yards connected by lines).

# Main evolutions since RSM 1.2

## Technical evolution: formal language

The formal language chosen by IRS30100 was UML. For semantic RSM, it is OWL (Web Ontology Language). Main reason is the ability to organise RSM into a federation of models, rather than an ever-growing set of UML packages.

OWL is designed with sharing models and data in mind, which technically solves the "federation" issue. In addition, OWL treats relations as first-class objects, unlike UML: this provides opportunities for re-engineering.

## Design evolution: becoming a reachability graph

The design intent of the original RTM model is clear: "navigability" expresses the possibility, for rolling stock, to move from a linear element to another one (without reversing direction of travel). Indeed, navigabilities could be exploited by software to answer queries such as "is there a path from A to B", but there were two shortcomings:

* path (node-edge-node-edge...) search needed to traverse reified edges; the graph was conceptually a line graph but was instantiated as a colored graph with two types of nodes (net elements and net relations);
* navigability relations (conceptually) were transitive at MICRO level, but no longer transitive as soon as details were lost (MESO and MACRO levels).

These issues could both be resolved by a re-designed topology that is fully compatible with the original one, but simpler and more expressive at the same time.

# Read more

See the wiki (under development).
