[![hackmd-github-sync-badge](https://hackmd.io/9MHp3vuPRVGy1Zf7VILBvw/badge)](https://hackmd.io/9MHp3vuPRVGy1Zf7VILBvw)

###### tags: `GTScript` `syntax` `workshop`

- **URL:** https://cscs.zoom.us/my/cscsvrlily
- **Next session:** TBA

[TOC]

:::info
## Issue description template

#### Change/Enhancement/Discussion: Short description

_Author:_ name (institution)

_Priority:_ :heavy_check_mark: / :heavy_minus_sign: / :x:

_Description:_ Here goes the description of the issue ….

_Proposals:_ Here goes the description of optional proposals (by the author or others) to discuss…
:::

## GTScript Syntax: Discussed Issues [2020/08/29]

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


## GTScript Syntax: Open Issues

### Field indexing

#### Discussion/Enhancement: Support for accessing fields with absolute and indirect indices
_Author:_ Rhea George (Vulcan)

_Priority:_

_Description:_ discuss if support for indirect and absolute indices should be added and for which axes (only `K` or also `I`,`J`). Indirect indices means that run-time variables can be used as field indices. Absolute indices means that index values are not longer interpreted as relative offsets to the current iteration point but as an absolute reference to an element of the grid.
 
Examples/use cases:
- Vulcan physics: ... 
```
def ...
```

_Proposals:_ it probably makes sense to restrict the discussion to the `K` axis, which is most common use case. Interesting points to discuss:
- Review use-cases and discuss the currently viable alternatives
- Implementation strategies for absolute/indirect access: complexity and performance penalty
- Strategies and consequences of for the analysis toolchain


#### Discussion/Enhancement: Support for vertical off-center writes
_Author:_ Rhea George (Vulcan), Oliver Fuhrer (Vulcan)

_Priority:_

_Description:_ some column based atmospheric science algorithms are intuitive to scientists to write in such a way that adjacent vertical layers are modified in conjunction. For example, if you take mass from the layer you are at and add it to the one below you, it is intuitive to write:
```
with computation(BACKWARD), interval(1, None):
        if mass[0, 0, 0] > 0:
             delta = x * y
             mass[0, 0, 0]  = mass[0, 0, 0]  - delta
             mass[0, 0, -1] = mass[0, 0, -1] + delta 
```
Without off-center writes, we need to write it something like
```
with computation(BACKWARD):
       with interval(-1, None):
               delta = x * y
               if mass > 0:
                     mass  = mass  - delta
        with interval(1, -1):
               delta = x * y
               if mass[0, 0, 1] > 0:
                     mass = mass + delta[0, 0, 1]
               if mass > 0:
                     mass  = mass  - delta
        with interval(0, 1):
               if mass[0, 0, 1] > 0:
                     mass = mass + delta[0, 0, 1]
```
This is not terrible, and makes sense with this small example, but it quickly can get confusing with more complex patterns.   The special handling at the top and bottom can be easy to forget to do.
Here is a sample we ported:    

Python loop version
```
for j in range(js, je ):
    for k in range(1, nz - 1):
        for i in range(is_, ie):
            if qv[i, j, k] < 0 and qv[i, j, k - 1] > 0.0:
                dq = min(
                    -qv[i, j, k] * dp[i, j, k], qv[i, j, k - 1] * dp[i, j, k - 1]
                )
                qv[i, j, k - 1] -= dq / dp[i, j, k - 1]
                qv[i, j, k] += dq / dp[i, j, k]
            if qv[i, j, k] < 0.0:
                qv[i, j, k + 1] += qv[i, j, k] * dp[i, j, k] / dp[i, j, k + 1]
                qv[i, j, k] = 0.0
```
Stencil version avoiding offcenter writes-introduce fields upper_fix and lower_fix:
```
@utils.stencil()
def fix_water_vapor_down(qv: Field, dp: Field, upper_fix: Field, lower_fix: Field):
    with computation(PARALLEL):
        with interval(1, 2):
            if qv[0, 0, -1] < 0:
                qv = qv + qv[0, 0, -1] * dp[0, 0, -1] / dp
        with interval(0, 1):
            qv = qv if qv >= 0 else 0
    with computation(FORWARD), interval(1, -1):
        dq = qv[0, 0, -1] * dp[0, 0, -1]
        if lower_fix[0, 0, -1] != 0:
            qv = qv + lower_fix[0, 0, -1] / dp
        if (qv < 0) and (qv[0, 0, -1] > 0):
            dq = dq if dq < -qv * dp else -qv * dp
            upper_fix = dq
            qv = qv + dq / dp
        if qv < 0:
            lower_fix = qv * dp
            qv = 0
    with computation(PARALLEL), interval(0, -2):
        if upper_fix[0, 0, 1] != 0:
            qv = qv - upper_fix[0, 0, 1] / dp
    with computation(PARALLEL), interval(-1, None):
        if lower_fix[0, 0, -1] > 0:
            qv = qv + lower_fix / dp
        upper_fix = qv
```
In the stencil version, it's hard to tell what's going on. The examples can get more complex. I think that unrestricted off-center writes might create a lot of problems, but this adjacent vertical adjustments pattern comes up a few times in the FV3 dycore and physics.

_Proposals:_ off-center writes could only be allowed in the vertical direction and only for sequential/non-parallel computations with any additional restrictions that'd need to be made.

---------------------------------------

### Computation

#### Change: Remove PARALLEL keyword as iteration order
_Author:_ Enrique G. P. (CSCS), Hannes Vogt (CSCS)
_Priority:_  

_Description:_ the `PARALLEL` keyword used as vertical iteration order in the computation definition is often misunderstood by users. We should find a cleaner way to specify stencils without a `FORWARD` or `BACKWARD` iteration order.

_Proposals:_ remove `PARALLEL` and define `computation()` as:

```python
def computation(iteration_order: Optional[IterationOrder] = None):
    pass
```
so it can be used with an empty `computation()` (or `None`?) :
```python
with computation():  # Ok
    ...
with computation(None):  # Ok ?
    ...
```

#### Change/Discussion: Specification order of vertical intervals
_Author:_ Till Ehrengruber (CSCS), Enrique G. P. (CSCS)

_Priority:_  

_Description:_ currently vertical intervals can be specified in any order inside a `computation()`, but they will be executed in the order defined by the `iteration_order` parameter, which means that reading GTScript code from top to bottom does not always reflect the execution order.

```python
with computation(BACKWARD):
  with interval(0, 1): # executed last
    ...
  with interval(-1, None): # executed first
    ...
  with interval(1, -1): # executed "inbetween"
    ...
```

_Proposals:_

 + [ ] Allow the user to specify in order of preference and reorder automatically (current behaviour in GT4Py)
      - :heavy_plus_sign: Boundary cases can be grouped together. Might be closer to how you think and develop the underlying algorithm
 + [ ] Force the user to specify in order of execution
      - :heavy_plus_sign: Comprehension of the execution order and hence the result of the computation is directly apparent

Both proposals are essentially trade-offs between the user who is writing the code and the user who is reading it (e.g. for modification).

#### Discussion/Enhancement: Expose the iteration index (_positional computations_)
_Author:_ Till Ehrengruber (CSCS), Enrique G. P. (CSCS)

_Priority:_  

_Description:_ users may want to access the current iteration index in the computation and use it as a numerical value. Since a stencil program will normally contain several iteration loops with different bounds (as part of the extended computation domain), if GT4Py exposes the iteration index, it should be clearly defined how the exposed index maps to the computation domain, and possible what are the possible consequences/interactions with the merging of stages   ofstages merging step in the analysis phase.

_Proposals:_
The syntax could be something like the following approach, but it must be always clear to the user which iteration loop is being uexposed:
```python
with computation(), interval(...):
    field_a = field_b[-2, 0, 0] + field_b[1, 0, 0]
    with axes(I) as i_index:  
        tmp = 3 * i_index
        field_c = 3 * i_indextmp * (field_a[-1, -1, 0] - field_a[1, 1, 0])
    out = field_c[-1, -1, 0] + field_c[1, 1, 0]
```


#### Enhancement: Definition of horizontal regions and named horizontal regions
_Author:_ Johann Dahm (Vulcan), Hannes Vogt (CSCS), Enrique G. P. (CSCS)

_Priority:_  

_Description:_ discuss GDP about horizontal region definitions and optional syntax to use custom names in the definition of the regions.

_Proposals:_ ...

---------------------------------------

### Grid and domain

#### Discussion: Definition of domain and extented domain concepts
_Author:_ Enrique G. P. (CSCS), Hannes Vogt (CSCS)

_Priority:_

_Description:_ discuss/review formal definitions of domain-related concepts. _WIP_

_Proposals:_ ...


#### Discussion: Use of grid objects in cartesian grid definitions
_Author:_ Till Ehrengruber (CSCS), Enrique G. P. (CSCS)

_Priority:_

_Description:_ discuss if a explicit grid object should be declared as a parameter of the definition function even for cartesian grids. _WIP_

_Proposals:_ ...


#### Discussion/Enhancement: Syntax to define objects in cartesian grid definitions
_Author:_ Enrique G. P. (CSCS)

_Priority:_

_Description:_ discuss syntax to define cartesian grids, optionally including axis splitter definitions. _WIP_

_Proposals:_ ...

