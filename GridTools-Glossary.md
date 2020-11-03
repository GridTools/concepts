### Boundary

### Cartesian Grid

A grid where neighbor [grid points](#grid-point) can be addressed by offsets (in 2 or 3 Cartesian dimensions). Typical examples are the lat-lon grid of COSMO or the cubed-sphere of FV3.

### Compute Domain

Describes a set of [grid points](#grid-point) on which the computation is applied. It doesn't contain grid points in the [halo](#halo) of a computation.

### Connectivity

### Extended Compute Domain

[Compute domain](#compute-domain) extended by halo points where intermediate result are computed in a staged computation.

### Grid point

### Halo

### ICON

A weather and climate model developed by DWD and MPI-M build on a icosahedral grid. ICON uses an [unstructured](#Unstructured-Grid), but the grid can also be represented as partially [structured](#Structured-grid), see [Pentagon problem](#pentagon-problem).

### Iteration Space

TODO disambiguate from compute domain or make it an alias.

### Local Field

An array located on a [grid point](#Grid-point) with one entry per neighbor location.

### Origin

### Pentagon problem

In the ICON grid, the sphere is built from rhomboids. Within each rhomboid the mesh and dual-mesh are fully structured (triangles, each vertex has exactly 6 edges). At corner points where the rhomboids are patched together, each vertex has only 5 neighbors. TODO add picture.

### Sparse Field

In the context of [unstructured grids](#Unstructured-Grid), a sparse field, is a field defined on the neighbors of each [grid point](#grid-point). The restriction of a sparse field to a single [grid point](#grid-point) is a [local field](#Local-Field). The name originates from the sparse nature of the second / neighbor dimension.

### Structured Grid

A grid where the [connectivity](#Connectivity) follows a regular pattern that allows strided access to neighbors. Examples of structured grids are [Cartesian grids](#Cartesian-Grid) or the structured representations of the ICON grid.

### Unstructured Grid

A grid without any assumptions on regularity of the [connectivity](#Connectivity). Note: any structured grid can be described by an unstructured grid.

## Language Generation

### Computation language

Tools like GT4Py generate stencils from DSL. The language in which they are generated (typically C++, CUDA) is called the computation language. This can also be a higher level language sometimes for debugging purposes. The Framework can but does not have to be mentioned when naming the computation language (for example "GridTools/C++", but not "GridTools" - that would be ambiguous).

### Language bindings

Code generation tools may also generate bindings and / or wrappers for using the stencils from an other (typically higher level) language. This is referred to as the tool "supporting language bindings for language X". The corresponding languages are "Languages with bindings support" or "bindings languages" for short.
