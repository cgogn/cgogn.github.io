---
title: CGoGN Documentation
id: documentation
---
# Implementation

CGoGN provides an efficient C++ implementation of oriented combinatorial maps. Its main features are combinatorial maps representation, manipulation, traversal and a dynamic attribute mechanism. It is also designed to take advantage of parallel architectures.

## Attribute containers

The central data structure used in CGoGN is **attribute containers**. An attribute container is a set of __attributes__ of arbitrary type that all have the same number of records. Attributes can be dynamically added and removed in an attribute container. An index identifies a tuple of records, one in each attribute of the container. Records can be dynamically added and removed in an attribute container. Removed entries are actually only marked as removed and skipped during the traversals. They are left available for further record additions.

Each __attribute__ is stored as a chunk array, i.e. a set of fixed size arrays. It allows the size of the attribute to grow while leaving all existing elements in place. Chunk size is chosen as a power of 2 so that the arithmetic operations needed to access the record of a given index (division and modulo) are very efficient.

A map in CGoGN is encoded as a set of attribute containers. The only mandatory container is the Dart container. According to the dimension of the map, it stores a variable number of attributes that encode the topological relations φ1, ..., φn. Each dart being represented by its index in the Dart container, these attributes store indices of linked darts in the Dart container. For each embedded cell dimension, the Dart container maintains an attribute that stores for each dart the index of its corresponding cell orbit. These indices correspond to entries in a dedicated attribute container. For example, in the map illustrated in Figure 5, Vertex and Face cells are embedded. The Dart container maintains _V_ and _F_ attributes that store the Vertex and Face index of each dart. Obviously, as explained in the [embedding](#embedding) section, all the darts of a same orbit share the same index. The Vertex and Face attribute containers store the data associated with each vertex and face cell of the map.

<p align="center">
	<img alt="Containers" src="/assets/img/containers.png">
	<em> <strong>Figure 5</strong>: Containers of a 2-dimensional combinatorial map with vertex and face embedding. The dart container stores φ1 and φ2 indices of linked darts and V and F indices of the associated vertices and faces attributes. </em>
</p>

## Basic types

CGoGN defines symbols in the `cgogn` namespace which will be omitted in the following.

The most basic type defined in CGoGN is `Dart` which stores an index in the Dart container.

An enumeration of orbits is also defined. These orbits correspond to the different cells that can be defined in a combinatorial map. They are defined as follows:
```c++
enum Orbit: uint32
{
    DART = 0,      // 1d vertex
    PHI1,          // 1d cycle, 2d face
    PHI2,          // 2d edge
    PHI21,         // 2d vertex
    PHI1_PHI2,     // 2d connected component, 3d volume
    PHI1_PHI3,     // 3d face
    PHI2_PHI3,     // 3d edge
    PHI21_PHI31,   // 3d vertex
    PHI1_PHI2_PHI3 // 3d connected component
};
```

The `Cell<Orbit>` template class stores a Dart that is publicly available as a `dart` property. These cell types can actually be seen as orbit-typed Darts.

A combinatorial map is composed of a Dart container and a container for each of the orbits defined in the above enumeration. These containers can be used to store the attributes associated to the cells of each embedded orbit. Containers corresponding to non-embedded orbits will remain unused during the lifetime of the map.

Several map types are defined (one for each dimension). For example, a 2-dimensional combinatorial map can be declared like that: `CMap2 map;`.

Each map type provides several convenience internal definitions for its cells types. For example a `CMap2` provides the following cell types: `Vertex`, `Edge`, `Face`, `Volume`. These cells are actually defined respectively as: `Cell<Orbit::PHI21>`, `Cell<Orbit::PHI2>`, `Cell<Orbit::PHI1>`, `Cell<Orbit::PHI1_PHI2>` which correspond to the orbits of these cells in a 2-dimensional map.

A `Vertex v` contains a Dart and can be thought of as the vertex of `v.dart`. Any other Dart of the same vertex orbit could equally represent the same cell.

## The Mesh abstraction

In a goal of genericity, all informations about a map class are obtain through a traits class `cgogn::mesh_traits<Mesh>`. Instead of using directly Maps class, all our code will use a template type parameter Mesh. A mesh can of course be a Map but also a object that represent a part of a map (CellFilter, CellCache as explained later), but also any type that could furnish all necessary constants, types and functions definitions.

The introduced complexiy could totally disappear from your code just by adding some type definitions in your class or function.  For example :

```c++
template <typename Mesh>
void my_algo(const Mesh& m)
{
    // cell definition
	using Vertex = typename mesh_traits<Mesh>::Vertex;
	using Face = typename mesh_traits<Mesh>::Face;
    // attribute definition
	template <typename T>
	using Attribute = typename mesh_traits<Mesh>::Attribute<T>;
```
Here we need to add the template keyword in each using definition, because we extract something from a template parameter.

In all examples of following sections will use these types definitions.

## Attributes

Attributes of any type can be added in a map. The `mesh_traits<Mesh>::Attribute<T>` template class provides a way to access to the type `T` values associated to the  cells of a map.
Some provided functions allow to *add*: `add_attribute<T,CELL>(mesh,att_name)`, *get* `get_attribute<T,CELL>(mesh,att_name)` or *remove* `remove_attribute<T,CELL>(mesh,att)` an attribute.
Functions to *add* and *get* return a `std::shared_prt<mesh_traits<Mesh>::Attribute<T>>`, we encourage you to use the `auto` C++ keyword for local declaration.

For example, a Vertex attribute storing an integer value on each vertex can be added in a Mesh (that could be a CMap2) like that (using previous type definitions):
```c++
using Mesh = CMap2;
using Vertex = typename mesh_traits<Mesh>::Vertex;
Mesh m;
...
auto attr = add_attribute<uint32, Vertex>(m, "attr");
```

An existing attribute can also be obtained by its name. As the requested attribute may not exist in the map, the validity of the attribute can then be verified:
```c++
using Mesh = CMap2;
using Face = typename mesh_traits<Mesh>::Face;
Mesh m;
...
auto area = get_attribute<double, Face>(m, "area");
if (area == nullptr)
{
    // there were no Face attribute of this type having this name in the map
}
```

Attribute values can be traversed globally using the range-based for loop syntax:
```c++
for (const double& v : *area)
{
    // do something with v
}
```
Remark: Here we have to used *area because the attribute's function return a shared_ptr<T>

Given a cell of the map, the value associated to this cell for an attribute can be accessed using the `value<T>(Mesh&, Attribute<T>&, Cell)` fonction. For example given to a mesh, an face Attribute of double and two Faces f1, f2,  we can write:
```c++
value<double>(m,area,f1) = 3.4;
double sum = value<double>(m,area,f1) + value<double>(m,area,f2);
```

Under the hood, this operator will first query the embedding index of the given cell and then access to the value stored at this index in the corresponding Cell attribute container. The embedding index of a cell does not depend on the accessed attribute. Here we have three attributes:
Warning when using this code we bypass the checking of the Cell on which attribute is attached.
```c++
uint32 findex = index_of(m,f);
(*attr1)[findex] = (*attr2)[findex] + (*attr3)[findex];
```

Attributes can also easily be removed:
```c++
remove_attribute(m,attr);
```

## Global traversals

The cells of the map can be traversed using the `foreach_cell` method. This method takes a callable that expects a `Cell<Orbit>` as parameter. It is the type of this parameter that determines the type of the cells that will be traversed by the foreach method. For example, the following code will traverse the vertices of the mesh m and call the given lambda expression on each vertex (we always use type definitions as shown in *Mesh abstraction*):
```c++
foreach_cell( m, [&] (Vertex v)
{
    // do something with v
});
```

No assumption can be made here about the `Dart` that represents each cell (e.g. `v.dart` in the previous example) when the processing function is called. Any dart of the same orbit could equally represent the same cell.

### Early stop

In order to stop the traversal before all the cells have been processed, the provided callable can return a boolean value. As soon as the callable returns false, the traversal is stopped. In the following example, we are looking for the first encountered vertex for which the Vec3 position value is on the negative side of the x=0 plan. If after the traversal, the declared Vertex is not valid, it means such a vertex has not been found in the map:
```c++
// given a mesh m
// given a Vertex looked_up_vertex ()
Vertex looked_up_v;
foreach_cell([&] (m, Vertex v) -> bool
{
    if (value<Vec3>(m,position,v)[0] < 0.0)
    {
        looked_up_v = v;
        return false;
    }
    return true;
});
if (!looked_up_v.is_valid())
{
    // no such vertex was found
}
```

### Parallelism

CGoGN can take advantage of parallel architectures to speed-up global traversals. A parallel traversal of the cells can be done using the `parallel_foreach_cell` function. The processing of the cells of the map will be spread among a number of threads that depends on the detected underlying hardware concurrency. In the following example, the displacement value value of each vertex is added to its position:
```c++
// given a mesh m
// given two Attribute<Vec3> position, displacement on VERTEX
parallel_foreach_cell(m, [&] (Vertex v)
{
    value(m,position,v) += value(m,displacement,v);
});
```

It is also possible to aggregate results coming from the parallel processing of several threads. In the following example, the average value of an Edge attribute is computed in parallel. Each thread accumulates the values and number of elements corresponding to the fraction of the mesh that it processes, and then the global result is computed:
```c++
// given a Mesh m
// given a attribute of type double on Edge length

std::vector<double> sum_per_thread(thread_pool()->nb_workers(), 0.0);
std::vector<uint32> nb_per_thread(thread_pool()->nb_workers(), 0);

parallel_foreach_cell(m, [&] (Edge e)
{
    uint32 thread_index = current_thread_index();
    sum_per_thread[thread_index] += length[e];
    nb_per_thread[thread_index]++;
});

double sum = 0.0;
for (double d : sum_per_thread)
    sum += d;
uint32 nb = 0;
for (uint32 n : nb_per_thread)
    nb += n;
double average = sum / double(nb);
```

<p class="warning"> Note that early stop is not available when doing parallel traversals. </p>

### Customized traversal

In their simplest form, the `foreach_cell` and the `parallel_foreach_cell` methods process all the cells of the mesh. But they can also be called with other kinds of parameters: a CellFilter<Mesh> and CellCache<Mesh>


#### CellFilter

A CellFilter encaspsule a Mesh object by adding a boolean functions on each type of cell you want to filer. Once CellFilter is created, it can be used as a normal mesh in a foreach.

In the following example, the function will print only the faces for which the area is above a given threshold:
```c++
void print_faces(const CMap2& m, const Attribute<double>& area, double threshold)
{
	CellFilter<CMap2> cfm(m);
	cfm.set_filter<Face>([&](Face f) -> bool {return (value<double>(cfm, area, f) > thresold;	});

	foreach_cell(cfm, [&](Face f) -> bool { 
		std::cout << f << std::endl;
		return true;
		});
}
```
A same CellFilter object can of course define one filter for each of its type of cell (for example Vertex, Edge, Face for CMap2)


<!-- ##### Cell filters

There are cases in which a function has to traverse several types of cells. Passing an additional Mask parameter for each type of cell would not be a very versatile solution. That is why the `foreach_cell` and `parallel_foreach_cell` methods can also accept an object derived from the `CellFilters` class as their second parameter.

Such an object defines a filtering function for each type of filtered cell. The following example shows a simple filtering object definition. The object is constructed given a map and a boolean vertex attribute. The filtered cells are here vertices for which the attribute is true and edges for which the attribute is true for its two incident vertices:
```c++
class SelectedCells : public CellFilters
{
public:
    SelectedCells(const CMap2& m, const VertexAttribute<bool>& s) :
        map_(m), selected_(s)
    {}

    inline bool filter(Vertex v) const
    {
        return selected_[v];
    }

    inline bool filter(Edge v) const
    {
        return selected[Vertex(v.dart)] && selected_[Vertex(map_.phi2(v.dart))];
    }

    inline uint32 filtered_cells() const
    {
        return orbit_mask<Vertex>() | orbit_mask<Edge>();
    }

private:
    const CMap2& map_;
    const VertexAttribute<bool>& selected_;
};
```

Such a filtering object can then be used like follows:
```c++
// given a CMap2 map
// given a VertexAttribute<bool> selected

template <typename MASK>
void f(const CMap2& map, const MASK& mask)
{
    map.parallel_foreach_cell(
        [] (Vertex v) { /* do something with v */ },
        mask
    );
    map.parallel_foreach_cell(
        [] (Edge e) { /* do something with e */ },
        mask
    );
}

SelectedCells sc(map, selected);
f(map, sc);
```

As you can see, an additional `filtered_cells` method is defined in the `CellFilters` objects. It is used by the `foreach_cell` or `parallel_foreach_cell` method to check that the requested cell traversal is actually filtered by the given object. If it is not the case, a warning is printed.

##### Cell traversors

Filtering functions are a great and versatile way to customize cell traversals. They are particularly well adapted to highly variable settings where the set of filtered cells changes regularly w.r.t. the filtering function. However, under the hood, the complete map is still traversed using classical algorithms that enumerate all the darts and use markers (a boolean attribute) to tag the darts of the processed orbits.

Other types of objects, derived from the `CellTraversor` class, can be used as a Mask and given as a second parameter to the `foreach_cell` or `parallel_foreach_cell` method. Any `CellTraversor` should provide a `begin<CellType>` and `end<CellType>` template methods that return an internal `const_iterator` type instance. These methods are then used by the `foreach_cell` or `parallel_foreach_cell` method to completely overload the classical traversal. Several `CellTraversors` are already provided in CGoGN.

__QuickTraversor__

The goal of a `QuickTraversor` is to avoid the enumeration of all the darts and the marking/unmarking cost related to the usage of a marker. To achieve this, it creates and maintains an Attribute of type Dart in the attribute container of the traversed cell type. Calling the `build<CellType>` method will fill this attribute with one representing dart per cell. When the `QuickTraversor` is given to the `foreach_cell` or `parallel_foreach_cell` method, its internal attribute is used to directly enumerate the cells without having to manage any marker, providing a significative speed-up.

However, in order to keep the `QuickTraversor` in sync with the map, special care must be taken when the connectivity is modified:
 - when a new orbit is created, one of its darts has to be written in the internal attribute. This can be done by passing the new cell to the `update` method.
 - when an existing orbit is modified, if one or more darts have been removed, the representing dart stored in the internal attribute may have been removed. A valid dart can be chosen again by passing the cell to the `update` method.
 - when an orbit is completely deleted, nothing has to be done as the attributes of this cell will not be traversed anymore.

In the following example, a `QuickTraversor` is built for the vertices of the map. Then an edge is cut, inserting a new vertex. In order for this new vertex to have a valid representing dart in the `QuickTraversor` internal attribute, this new vertex is updated:
```c++
// given a CMap2 map
QuickTraversor qtrav(map);
qtrav.build<Vertex>();
map.foreach_cell(
    [&] (Vertex v) { /* do something with v */ },
    qtrav
);
// given a Edge e
Vertex v = map.cut_edge(e);
qtrav.update(v);
```

The selection of the dart that represents each cell can be customized by giving an additional function as parameter to the `build` and `update` methods. This function should take a Cell as parameter and return a Dart. This customization is useful if you want to be able to make assumptions about the dart that represents each cell when it is given to the processing function during a traversal.

__FilteredQuickTraversor__

A `FilteredQuickTraversor` is a variation of a `QuickTraversor`. In this modified version, the internal attribute is still filled with one dart per cell when calling the `build` method and the same updates have to be done in order to keep it in sync with the map. The difference is a new `set_filter<CellType>` method that provides a filter which is evaluated on-the-fly each time the `FilteredQuickTraversor` is used for the traversal of the cells. It can be seen as a combination of the efficiency of the traversal without marking and the versatility of the on-the-fly cells filtering.

The dart selection customization is also available by giving a dart selection function as an additional parameter to the `build` and `update` methods. -->

#### CellCache

With the previous Masks, even when using a filtering function, all the cells of the map are always traversed and the filtering function is evaluated on-the-fly for each cell in order to decide if the processing function should be called on it or not. This can be great in a dynamic environment where the set of filtered cells changes regularly. But if the set of filtered cells is stable, and moreover if this set is small w.r.t. the number of cells of the map, a more efficient solution can be proposed.

A `CellCache` can store a set of cells for each type of cell. These sets will be directly traversed by the `foreach_cell` or `parallel_foreach_cell` method.
The most easy way to populate a cache is to use the build method `build<CellType>` which builds the set of the mentioned cell type, using the filtering bool function parameter.
There is also an `add<CELL>` method for individual insertion, and a global clear() method.

In the following example, all the degree 5 vertices are put into a cache which is then used in a traversal that is efficient and restricted to those vertices:
```c++
// given a CMap2 map
CellCache cache(map);
cache.build<Vertex>([&] (Vertex v)
{
    return degree(map,v) == 5;
});

foreach_cell(cache, [&] (Vertex v)
{ /* do something with v */ 
});
```

<p class="warning"> Note that if a new cell that satisifies the filter is inserted into the map, it will not be part of a cache that has already been built. </p>

<p class="warning"> Note also that if any cell stored in the cache is deleted from the map, the cache is invalid and any subsequent traversal with this cache will fail. </p>

The "snapshot" property of the `CellCache` can be particularly exploited within algorithms that insert new cells during the traversal of the map. Indeed, without such a mechanism, the newly inserted cells could also be traversed resulting in a possible infinite loop. In the following example, all the edges of the map are cut by inserting a new vertex. Only the edges that existed at the moment the cache was built are considered:
```c++
// given a CMap2 map
CellCache cache(m);
cache.build<Edge>();
foreach_cell(cache, [&] (Edge e) { cut_edge(m,e); });
``` 

## Local traversals

All the __incidence__ and __adjacency__ relations between cells are encoded within a combinatorial map. We first give an intuitive definition of what we mean by incidence and adjacency relations and then show how to make these neighborhood traversals in CGoGN.

### Incidence

Let C1 and C2 be two cells of different dimensions with dim(C1) > dim(C2). C2 is said to be __incident__ to C1 if it belongs to the sub-tree under C1 in the incidence graph (for example, the four vertices and the four edges that bound a quad face in a 2D mesh are said to be incident to this face). C1 is said to be __incident__ to C2 if it is part of the ancestors of C2 in the incidence graph (for example, the six edges and the six faces around a vertex in a regular 2D triangle mesh are said to be incident to this vertex).

The following Figure 6 illustrates the incident cells in a 2-dimensional map and Figure 7 illustrates some of the incident cells in a 3-dimensional map.

<p align="center">
	<img alt="local traversals" src="/assets/img/traversor2XY.png">
	<em> <strong>Figure 6</strong>: Incident cells traversals in a 2-dimensional map. Central cell is drawn in green and incident cells are drawn in red. Top-left: incident edges of a vertex; Bottom-left: incident faces of a vertex; Top-middle: incident vertices of an edge; Bottom-middle: incident faces of an edge; Top-right: incident vertices of a face; Bottom-right: incident edges of a face. </em>
</p>

<p align="center">
	<img alt="local traversals" src="/assets/img/traversor3XY.png">
	<em> <strong>Figure 7</strong>: Some of the incident cells traversals in a 3-dimensional map. Central cell is drawn in green and incident cells are drawn in red. From left to right: incident edges of a vertex and incident faces of a vertex; incident volumes of an edge; incident volumes of a face; incident edges of a volume. </em>
</p>

### Adjacency

Two cells of equal dimension C1 and C2 are said to be __adjacent__ if they share a common incident cell. For example: two vertices are said to be __adjacent__ _through an edge_ if they are both incident to a same edge; two faces are said to be __adjacent__ _through a vertex_ if they are both incident to a same vertex.

<p align="center">
	<img alt="local traversals" src="/assets/img/traversor2XXaY.png">
	<em> <strong>Figure 8</strong>: Adjacent cells traversals in a 2-dimensional map. Central cell is drawn in green and adjacent cells are drawn in red. Top-left: adjacent vertices through an edge; Bottom-left: adjacent vertices through a face; Top-middle: adjacent edges through a vertex; Bottom-middle: adjacent edges through a face; Top-right: adjacent faces through a vertex; Bottom-right: adjacent faces through an edge. </em>
</p>

### Local traversal methods

CGoGN provides methods to traverse all these local neighborhoods. These methods all work in the same way: the callable given as second parameter is called on all the requested incident or adjacent cells of the cell given as first parameter. If this callable returns a boolean value, the traversal is stopped as soon as this value is false.

In a 2-dimensional map, the local neighborhood traversal functions are the following:
 - `foreach_incident_edge(Mesh, Vertex, [] (Edge) {})`
 - `foreach_incident_face(Mesh, Vertex, [] (Face) {})`
 - `foreach_adjacent_vertex_through_edge(Mesh, Vertex, [] (Vertex) {})`
 - `foreach_adjacent_vertex_through_face(Mesh, Vertex, [] (Vertex) {})`
 - `foreach_incident_vertex(Mesh, Edge, [] (Vertex) {})`
 - `foreach_incident_face(Mesh, Edge, [] (Face) {})`
 <!-- - `foreach_adjacent_edge_through_vertex(Mesh, Edge, [] (Edge) {})` -->
 <!-- - `foreach_adjacent_edge_through_face(Mesh, Edge, [] (Edge) {})` -->
 - `foreach_incident_vertex(Mesh, Face, [] (Vertex) {})`
 - `foreach_incident_edge(Mesh, Face, [] (Edge) {})`
 <!-- - `foreach_adjacent_face_through_vertex(Face, [] (Face) {})` -->
 <!-- - `foreach_adjacent_face_through_edge(Mesh, Face, [] (Face) {})` -->
 - `foreach_incident_vertex(Mesh, Volume, [] (Vertex) {})`
 - `foreach_incident_edge(Mesh, Volume, [] (Edge) {})`
 - `foreach_incident_face(Mesh, Volume, [] (Face) {})`

The following example illustrates a function that computes the average of vertex attribute values over the vertices incident to a face in a 2-dimensional map:
```c++
template <typename T>
T average(const Mesh& map, Face f, const Attribute<T>& attribute)
{
    T result(0);
    uint32 nbv = 0;
    foreach_incident_vertex(m, f, [&] (Vertex v)
    {
        result += attribute[v];
        ++nbv;
    });
    return result / nbv;
}
```
Of course here the type T must be numerical and provide operators += and /.


Given that all the equivalent neighborhood traversal functions are also defined in 3-dimensional maps, the previous function can be easily generalized to the computation of the average of vertex attribute values over the vertices incident to a n-cell in a n-dimensional map:
```c++
template <typename T, typename CellType, typename MESH>
T average(const MESH& m, CellType c, const typename Attribute<T>& attribute)
{
    T result(0);
    uint32 nbv = 0;
    foreach_incident_vertex(m, c, [&] (typename MESH::Vertex v)
    {
        result += value(m,attribute,v);
        ++nbv;
    });
    return result / nbv;
}
```

## Properties

Several functions can provide some properties about the map or the cells.

 - nb_cells
 <!-- - nb_boundaries
 - same_cell -->
 - degree
 - codegree
 - is_incident_to_boundary
 <!-- - is_adjacent_to_boundary -->

## Operators
Each of the following operator is provided only if the *Mesh* type that allows it. For example the *flip_edge* operator has sense only on surfacic Mesh like CMap2
- Vertex cut_edge(Mesh,Edge) 
- Vertex collapse_edge(Mesh,Edge)
- void flip_edge(Mesh,Edge)
- void merge_incident_faces(Mesh,Edge)
- Edge cutFace(Mesh)
- Face cut_volume(Mesh,std::vector<Dart>)

**Warning**: Operators do not fill or compute any attribute even the position.
You can add the false as last parameter on each function if you want apply pure topological operators (embedding indices not updated)

## Creation
- Face add_face(Mesh, uint32)
- Volume add_prism(Mesh, uint32)
- Volume add_pyramid(Mesh, uint32)
**Warning**: Operators do not fill or compute any attribute even the position.
You can add the false as last parameter on each function if you want apply pure topological operators (embedding indices not updated)

## Marking

Ther are some classic problems like:
- how be sure to not traverse twice the same cell.
- how do no traverse the cells that I am currently adding in the mesh

To avoid these problems we use Markers. CellMarkers for cells and DartMarker for low-level usage on maps

### CellMarker

A CellMarker if an object that allow to mark/unmark cells of a choosen type (second template parameter), to test if the cell is marked, and clean 

```c++
// creation (here for vertices)
CellMarker<Mesh,Vertex> cm{m};

// test if a cell v is maked
if (!cm.is_marked(v))

// mark the cell	
cm.mark(v)

// unmark the cell
cm.unmark(v);

// possible to unmark all marked cell (call implicitly on destruction of CellMarker)
cm.unmark_all();
```

You can use as many markers as you want, like color markers on a white table.

A CellMarker use an attribute for storing its marking data. That could be totally unoptimal if you know that your are marking very few cells, because the cost of cleanning of the marker is
linked to the size of the mesh. For these cases it is advise to use a `CellMarkerStore`. The interface is the same, but is store the marked cells are also stored in a vector, that allow to clean the marker very quickly. Warning unmark if here more costly than mark (find algorithm in a vector). You have also access to the vector of cells

Use only CellMarkerStore if:
- you mark few cells (compare to number of cells of the mesh)
- you do a lot of mark and unmarkall but very few unmark (individual cells cells)
- you do a lot unmarkall 
- you do very few individual unmark

### Dart marking (for low level programming on CMap)



