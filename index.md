## Introduction

A __mesh__ is the cellular decomposition of geometric objects such as curves, surfaces or volumes.

Mesh data structures are of fundamental importance in many fields such as surface modeling, mesh generation, finite element analysis, physical simulation, geometry processing, visualization or computational geometry.

__Topological models__ provide neighborhood relations between the cells of the decomposition (vertices, edges, faces, volumes). This information is mandatory for many algorithms in the mentioned application fields.

The data structure should:
 - provide an easy and efficient way to traverse the cells
 - provide an easy and efficient way to traverse local neighborhoods
 - allow to store data with the cells
 - provide operators to modify the connectivity
 - remain efficient both in memory & time even in complex highly dynamic settings

Many existing mesh data structures actually find their roots in the notion of __combinatorial map__ that is is a mathematically
defined structure that represents the subdivision of n-dimensional manifolds.

## Combinatorial maps

Combinatorial maps are dimension-independent and rely on a single element along with a simple set of relations. All the information about the cells and their incidence and adjacency relations is contained within this simple model.

All neighborhood traversals are resolved in optimal time (linear in the
number of traversed cells) without having to maintain any additional information.

As in other topological models, there is a strong separation between the topological structure of the subdivision and the attributes that can be associated to the cells.

## Implementation

