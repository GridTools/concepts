###### tags: `GTScript` `syntax` `workshop`

- **Date:** 2020/08/19, from 16:00 - 18:00 (CEST)
- **URL:** https://cscs.zoom.us/my/cscsvrlily

[TOC]


## GTScript Syntax: Discussed Issues

### Data types

#### Discussion: Definition of supported data types in GTScript

_Author:_ Enrique G. P. (CSCS)

_Priority:_

_Description:_ there has not been discussion about the supported data types in GTScript stencils. This is not a big issue for `Storage` objects since they can deal with NumPy `dtypes` transparently, but a clear definition of which ones are supported would be helpful for the users and needed anyway for analysis and code generation tasks.

_Proposal:_
- Promote the list of the currently supported data types in the IR definition as the standard data types for GTScript (`bool`, `int8`, `int16`, `int32`, `int64`, `float32`, `float64`) and define a list of data types which might be interesting to add (or remove) in the future.
- (Optional) Interaction with Dusk/Dawn toolchain?

_Decision:_ **Approved**. Interesting data types to be added or discussed in the future are `unsigned` and `complex` types with different precision.


#### Enhancement: Definition of typed scalars (externals and literals)
_Author:_ Enrique G. P. (CSCS)

_Priority:_

_Description:_ data types are currently specified (both for fields and scalar parameters) in the type annotations of the definition function:
```python
@gtscript.stencil(...)
def my_stencil(field_a: Field[float], field_b: Field[float], value: float):
    pass
```

The content of each annotation sets argument's data type following these rules:

- if the annotation is a Numpy dtype-like instance, it will be used as it is
- if the annotation is a builtin numeric type (`float`, `int`) it will be first transformed to a NumPy dtype using NumPy rules (which means: `float` -> `np.float64` and `int` most likely `np.int64`)
- if the type annotation is a string, the actual data type will be the value of the string in the `dtypes` mapping provided as argument in the stencil decorator call. Example:
```python
@gtscript.stencil(..., dtypes={"small_float": np.float32, "large_float": np.float64})
def my_stencil(field_a: Field["small_float"], field_b: Field["large_float"], value: np.float64):
    pass
```

Different approaches can be mixed freely in the same definition signature, and data type aliases are allowed, therefore the following example is also valid:
```python
my_float_type = np.float64

@gtscript.stencil(..., dtypes={"my_float": np.float32})
def my_stencil(field_a: Field[my_float_type], field_a: Field["my_float"], value: float):
    pass
```

These options allow the user to write "generic" stencil definitions where the actual data types are choosen at compile time (the `stencil` decorator call). However, since there is no way to define a scalar literal (or a `external` constant) with a specific data type in GTScript, applying these rules to scalar values means that literals are always promoted to `int64` or `float64` data types in the current implementation. This hidden type promotion can lead to generated code with unneccessary type casting operations and numerical issues, since some operations might be executed at the wrong precision.

_Proposal:_ extend the syntax to specify the data types of all the values used in GTScript code.
- For externals: support and document the use of NumPy scalars (https://numpy.org/doc/stable/reference/arrays.scalars.html) to pass external constant values

Example: 
```python
ALPHA = 4.5

@gtscript.stencil(backend=backend, externals={"ALPHA": np.float32(ALPHA)})
def my_stencil(...):
    pass
```

- For literals, different _non-exclusive_ strategies:
  1. Inline creation of typed literals (so it can be used inside expressions): `float32(4.5)`
  2. Provide a `cast_to()` operator for dtypes defined with strings: `field_b = field_a + cast_to("small_float", 43.5)`
  3. Type annotations for symbol definitions (it should also work with string annotations for types provided at compilation time): `ALPHA: float32 = 4.5`,  `ALPHA: "my_float_type" = 4.5`
  4. Use `dtypes` argument of `gt.stencil`, with the builtins `int` and `float` as keys, to define the actual implementation data types of Python builtin dtypes (for compile-time definition of the actual dtypes, instead of hardcoding them in the definition code): 
```python
@gtscript.stencil(dtypes={int: np.int32, float: np.float32})
def my_stencil(...):
  field_a = field_a + 43.5 # => field_a = field_a + float32(43.5)
```
  5. Use user-defined type aliases as keywords for casting: `small_float(43.5)`

_Decision:_ As a first step, we will implement the following options: 1, 3, 4, and later decide about 5.

_Follow-up questions:_
- Where are type annotations required? Currently not required on functions, but when stencils can call other stencils this will not be parallel.

---------------------------------------

### Field definitions

#### Discussion: Clarification of the _Field_ concept

_Author:_ Enrique G. P. (CSCS), Till Ehrengruber (CSCS), Hannes Vogt (CSCS)

_Priority:_

_Description:_ before discussing specific syntatic proposal for the definition of fields in GTScript, it would be useful to clarify the concept of Field and its link with the domain/grid. The main open question is how to classify the dimensions of a field (domain dimensions, data dimensions, ...) and how to map them to the domain.

_Proposal:_ The following table sketches a possible definition of a Field as a mapping from some set of elements in the domain to scalar or vector quantities, and the representation of a similar mapping concept in Python via type hints. Following this idea, the left-hand side of the mapping always represent _domain dimensions_, and the right-hand side _data dimensions_, and it is then clear that any high-dimensional field (dims > 3) includes additional data dimensions.

| Meaning                     | Math                                              | Python/GTScript           |
| --------------------------- |-------------------------------------------------- | ------------------------- | 
| Mapping/function            | $f: X \rightarrow Y$                              | `Callable[[X], Y]  ## Mapping[X, Y]`    | 
| Cartesian 3D                | $F: I \times J \times K \rightarrow \mathbb{R}$   | `Field[[I, J, K], float]` | 
| Cartesian 2D                | $F: I \times J \rightarrow \mathbb{Z}$            | `Field[[I, J], int]`      | 
| Cartesian 3D (vector field) | $F: I \times J \times K \rightarrow \mathbb{R}^3$ | `Field[[I, J, K], Tuple[float, float]]` | 

This proposal also works for unstructured meshes:

| Meaning                                     | Math                                              | Python/GTScript          |
| ------------------------------------------- | ------------------------------------------------- | ------------------------ | 
| Field on vertices                           | $F: V \rightarrow \mathbb{R}$                     | `Field[[Vertex], float]` |
| Field on vertex->edge ("sparse")            | $F: V \times E^{adj} \rightarrow \mathbb{R} \\ v \in V, e \in E^{adj} \iff e \text{ is adj. to } v$ | `Field[[Vertex, Adjacent[Edge]], float]`  | 
| Alt:  Field on vertex->edge ("local field") | $F: V \rightarrow (G: E^{adj} \rightarrow \mathbb{R}) \\ v \in V, e \in E^{adj} \iff e \text{ is adj. to } v$ | `Field[[Vertex], LocalField[[Edge], float]]`  | 


#### Enhancement: Enhanced definition of fields for cartesian fields
_Author:_ Enrique G. P. (CSCS), Till Ehrengruber (CSCS), Hannes Vogt (CSCS)

_Priority:_

_Description:_ discuss a syntax to support the definition of lower (1d, 2d) dimensional (and optionally higher dimensional) fields for cartesian grids.

_Proposal:_ following the interpretation of the Field concept as a mapping, and since the Python type hint for functions and callables is `Callable[[args], result]`, this is a possible way to define fields in a sufficiently flexible way, while still keeping readiblity (all keywords are assumed to live in the `gtscript` module). The general formula is: `Field[[domain_elements], data_item]`, with specific adaptations to more complicated cases.

Examples:
- 3d field on cell centers: `Field[[I, J, K], float32]`
- 2d horizontal field on cell centers: `Field[[I, J], float32]`
- 1d vertical field on cell centers: `Field[[K], float32]`
- 3d field on J cell edges (both):
    + [ ] `Field[[I, J[LEFT, RIGHT], K], float32]`
    + [ ] `Field[[I, J_INTERFACES, K], float32]`
    + [ ] `Field[[I, J ^ INTERFACES, K], float32]`
    + [ ] `Field[[I, J | INTERFACES, K], float32]`
    + [ ] All of the previous, but with `North`, `East`, `South`, `West`
    + [ ] Others ...
- 3d field on J left cell edges:
    + [ ] `Field[[I, J[LEFT], K], float32]`
    + [ ] `Field[[I, J_LEFT, K], float32]`
    + [ ] `Field[[I, J ^ LEFT, K], float32]`
    + [ ] `Field[[I, J | LEFT, K], float32]`
    + [ ] `Field[[I, J_LEFT_INTERFACE, K], float32]`
    + [ ] `Field[[I, J_INTERFACES[LEFT], K], float32]`
    + [ ] Avoid horizontal axes altogether and use offsets to reference neighbors (just something to ponder)
    ```python
    def stencil(f_a : Field[[Edge, K], float32], f_b : Field[[Cell, K], float32])
      with location(Cell), ...:
        # since f_a is a field on edges [-1, 0, 0] refers to the left edge
        #  of the currently considered cell. [1, 0, 0] would be the right edge
        #  and [0, 0, 0] is invalid
        f_a[-1, 0, 0] = f_b[0, 0, 0]
    ```
    + [ ] Others ...

In the case of high dimensional fields with data dimensions, it should be first discussed if the sizes of the dimensions should be fixed (and thus the sizes are part of the type) or if they can be flexible (and then specified only at compile time or run time). 

Examples:
- 5d field => a 3d field of matrices (M x N) on cell centers:  
  (should or could `M`, `N` be fixed values?)
    + [ ] `Field[[I, J, K], Matrix[float32]`
    + [ ] `Field[[I, J, K], Tensor[2, float32]]`
    + [ ] `Field[[I, J, K], Tensor[[M,N], float32]`
    + [ ] `Field[[I, J, K], Matrix[[M, N], float32]]`
    + [ ] `Field[[I, J, K], Vector[N, float32]]`
    + [ ] Allow both, fixed and dynamically sized: `Matrix[float32]`, `SMatrix[[M, N], float32]`
    + [ ] Others ...
- 4d field => a 3d field of N-dimensional row-vectors:
    + [ ] `Field[[I, J, K], Vector[N, float32]]`
    + [ ] `Field[[I, J, K], Tensor[[N], float32]`
    + [ ] Others ...  
  Note: `Matrix[1, N]` has different meaning.

_Decision:_ Accept trivial extension to field annotations (structured grids) and delay decision on edges and non-scalar data types until later.


#### Enhancement: Definition of fields for unstructured meshes
_Author:_ Enrique G. P. (CSCS), Till Ehrengruber (CSCS)

_Priority:_  

_Description:_ discuss a syntax to support the definition of fields for unstructured meshes, taking into account sparse dimensions/neighbor chains.

_Proposal:_ following the same proposal of the enhanced field definition for cartesian grids (in order to be compatible with it), a possible way to define fields for unstructured meshes would be:

Examples:
- field on vertices: `Field[[Vertex], float]`
- field on edges: `Field[[Edge], float]`
- field on cells: `Field[[Cell], float]`

**_Sparse_ concept for neighborhood chains:**
- field on edges adjacent to cells (_sparse_ concept):
    + [ ] `Field[[Edge, Adjacent[Vertex]], float]`
    + [ ] `Field[[Cell, Neighbor[Edge]], float]`
    + [ ] `Field[[Cell >> Edge], float]`
    + [ ] `Field[[Cell ^ Edge], float]`
    + [ ] `Field[[Cell @ Edge], float]`
    + [ ] Others ...

- field on vertices adjacent to edges adjacent to cells (_sparse_ concept):
    + [ ] `Field[[Cell, Adjacent[Edge, Vertex]], float]`
    + [ ] `Field[[Cell, Neighbor[Edge, Vertex]], float]`
    + [ ] `Field[[Cell, Adjacent[Edge | Vertex]], float]`
    + [ ] `Field[[Cell, Neighbor[Edge | Vertex]], float]`
    + [ ] `Field[[Cell >> Edge >> Vertex], float]`
    + [ ] `Field[[Cell ^ Edge ^ Vertex], float]`
    + [ ] `Field[[Cell @ Edge @ Vertex], float]`
    + [ ] Others ...

**_Local field_ concept for neighborhood chains:**
- field on edges adjacent to cells (_local field_ concept): `Field[[Cell], LocalField[[Edge], float]]`
- field on vertices adjacent to edges adjacent to cells (_local field_ concept):
    + `Field[[Cell], LocalField[[Edge, Vertex], float]]`
    + [ ] Others ...

_Decision:_ More examples and discussions are needed before taking a decision.


### Field indexing

#### Enhancement: Enhance syntax for field indexing in cartesian grids
_Author:_ Enrique G. P. (CSCS), Oliver Fuhrer (Vulcan) 

_Priority:_

_Description:_ index operations with non-3d fields can be quickly become ambiguous or ill-defined, and lack of direction/axes variables end up involving a lot of code duplication repeating exactly the same statements several times, one per axes
 
_Proposal:_  
- Support variables containing axes:
```python
dir = I
field[dir + 3]

def gtscript_function(field, dir):
    return field[dir + 3] + 5.0
```
- Indexing domain dimensions:  
  (examples for `field: Field[[I, K], float32] => [I-1, J=, K+1]`)
    + [ ] Map indices to dimensions according to the Field declaration: `field[-1, 1]`
         - :heavy_check_mark: succint
         - :x: error-prone
    + [ ] Always specify the 3 domain indices in order: `field[-1, False, 1]`
         - :heavy_check_mark: not ambiguous
         - :heavy_minus_sign: strict I, J, K ordering
         - :x: verbose
    + [x] Explicit axes for non-zero offsets in any order: `field[K+1, I-1]`
         - :heavy_check_mark: succint
         - :heavy_check_mark: readable
         - :heavy_minus_sign: unusual
    + [ ] Others ...

- Indexing data dimensions:  
  (examples for `field: Field[[I, K], Vector[2, float32]] => [I-1, J=, K+1]`)
    + [ ] Use `__getitem__` of the indexed point: `field[K+1 | I-1][1]`
    + [ ] Use `__getitem__` of the field: `field[K+1 | I-1, 1]`
    + [ ] Others ...

_Decision:_
  - Support variables containing axes: **approved**
  - Indexing domain dimensions: **Explicit axes for non-zero offsets in any order** (taking into account collisions with variables declarations and with the syntax for absolute indexing)


#### Enhancement: Definition of syntax to specify temporary fields dimensionality
_Author:_ Oliver Fuhrer (Vulcan), Enrique G. P. (CSCS)

_Priority:_

_Description:_ currently GT4Py infers the type of temporaries from the resulting type of the RHS of the assignement to the temporary. For large stencils with many input / output fields of different type (`real`, `integer`, `boolean`), it feels that we are loosing type checking by not being explicit about the type of temporaries. 


Also, in operations mixing fields of diferent dimensionality/rank (e.g. 3d and 2d fields), it is not always clear the shape of the output result and it would likely need to be explicitly declared by the user. For example:
```python
c = a[i, j] * b[k]
```
could imply `c[i, j, k]` or simply `c[i, j]`.
 
**Data dimensions:** if it is allowed to define temporary fields with extra data dimensions, the ambiguities could be harder to solve automatically without explicit user annotations (examples missing).


_Proposals:_
- [ ] Use type annotations in the declaration of temporary fields:
```python
c: Field[I, J, K] = a[i, j] * b[k]
```
- [x] Define that all temporary fields always behave as 3d fields, and implement an analysis step to reduce the actual rank of the temporary in the generated code when this is possible.

_Decision:_ as a first step, we will try to implement an analysis step to optimize the rank of temporaries without requiring the user to provide type annotations, but this decision could be reverted in the future if we find corner cases where it is not possible to select the right shape without additional information provided by the user.

