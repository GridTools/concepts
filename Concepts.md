# Glossary for GridTools related projects

### Boundary

### Cartesian grid

### Compute domain

### Halo

### Iteration space

### Origin

### Unstructured

## Language Generation

### Computation language

Tools like GT4Py generate stencils from DSL. The language in which they are generated (typically C++, CUDA) is called the computation language. This can also be a higher level language sometimes for debugging purposes. The Framework can but does not have to be mentioned when naming the computation language (for example "GridTools/C++", but not "GridTools" - that would be ambiguous).

### Language bindings

Code generation tools may also generate bindings and / or wrappers for using the stencils from an other (typically higher level) language. This is referred to as the tool "supporting language bindings for language X". The corresponding languages are "Languages with bindings support" or "bindings languages" for short.
