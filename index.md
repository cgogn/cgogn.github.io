---
title: CGoGN Home
id: home
---
CGoGN is develop by the research team [IGG](https://igg.icube.unistra.fr/index.php/Accueil) of the [ICube](https://icube.unistra.fr/) Laboratory of the University of Strasbourg
It is now part of a most generic platform name [GAIA](https://gaia.icube.unistra.fr/index.php/Accueil)  (in a pole name Modelling & Simulation) which aim to treat most generic type of data, from analysis until Visualisation.

CGoGN is a geometric modeling C++ library that provides an efficient implementation of combinatorial maps.

One of the fundamental idea of topological models is the strong separation between:
- the description of the topology of the mesh (the cells and their adjacency relationships)
- the space in which the cells are embedded (attributes associated to the cells)

Following this fundamental principle, our implementation also aims at:
- efficiency (more contiguous tables, less pointers)
- low memory consumption
- genericity (models, dimensions)
- ease of use (more functional, less imperative)

You can jump [here]({{ site.baseurl }}{% link combinatorial_maps.md %}) to read an introduction to meshes and combinatorial maps. Depending on your current understanding of this model and on the level on which you plan to use the library, you may or may not need to read it. So feel free to jump directly to the [documentation]({{ site.baseurl}}{% link documentation.md %}).

# Introduction

A __mesh__ is the cellular decomposition of geometric objects such as curves, surfaces or volumes. Mesh data structures are of fundamental importance in many fields such as surface modeling, mesh generation, finite element analysis, physical simulation, geometry processing, visualization or computational geometry.

__Topological models__ provide neighborhood relations between the cells of the decomposition (vertices, edges, faces, volumes). This information is mandatory for many algorithms in the mentioned application fields.

The data structure should:
 - provide an easy and efficient way to traverse the cells
 - provide an easy and efficient way to traverse local neighborhoods
 - allow to store data with the cells
 - provide operators to modify the connectivity
 - remain efficient both in memory & time even in complex highly dynamic settings

Many existing mesh data structures actually find their roots in the notion of __combinatorial map__, a mathematically defined structure that represents the subdivision of n-dimensional manifolds.
