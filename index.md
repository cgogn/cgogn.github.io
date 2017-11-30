---
title: CGoGN Home
id: home
---

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
