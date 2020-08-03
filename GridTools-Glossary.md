### Boundary

### Cartesian Grid

A grid where neighbor [grid points](#grid-point) can be addressed by offsets in 2 or 3 Cartesian dimensions. Typical examples are the lat-lon grid of COSMO or the cubed-sphere of FV3.

### Compute Domain

Describes a set of [grid points](#grid-point) on which the computation is applied. It doesn't contain grid points in the [halo](#halo) of a computation.

### Extended Compute Domain

[Compute domain](#compute-domain) extended by halo points where intermediate result are computed in a staged computation.

### Grid point

### Halo

### Iteration Space

TODO disambiguate from compute domain or make it an alias.

### Origin

### Structured Grid

A grid where cells follow a regular pattern that allows strided access to neighbors. Examples of structured grids are [Cartesian grids](#Cartesian-Grid).

### Unstructured Grid

A grid without any assumptions on regularity. Note: any structured grid can be described by an unstructured grid.

## Language Generation

### Computation language

Tools like GT4Py generate stencils from DSL. The language in which they are generated (typically C++, CUDA) is called the computation language. This can also be a higher level language sometimes for debugging purposes. The Framework can but does not have to be mentioned when naming the computation language (for example "GridTools/C++", but not "GridTools" - that would be ambiguous).

### Language bindings

Code generation tools may also generate bindings and / or wrappers for using the stencils from an other (typically higher level) language. This is referred to as the tool "supporting language bindings for language X". The corresponding languages are "Languages with bindings support" or "bindings languages" for short.
