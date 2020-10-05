###### tags: `GTScript` `syntax` `workshop`

- **Date:** 2020.09.29 at 16:30 - 18:00 (CEST)

[TOC]


## GTScript Syntax: Discussed Issues [2020/09/29]

### Field indexing

#### Discussion/Enhancement: Support for accessing fields with absolute and indirect indices
_Author:_ Rhea George (Vulcan)

_Priority:_

_Description:_

Discuss if support for indirect and absolute indices should be added and for which axes (only `K` or also `I`,`J`). Indirect indices means that run-time variables can be used as field indices. Absolute indices means that index values are not longer interpreted as relative offsets to the current iteration point but as an absolute reference to an element of the grid.
 
Examples/use cases:
- Vulcan physics: ... 
```
def ...
```

_Proposals:_ it probably makes sense to restrict the discussion to the `K` axis, which is most common use case. Interesting points to discuss:
- Review use-cases and discuss the currently viable alternatives
- Implementation strategies for absolute/indirect access: complexity and performance penalty
- Strategies in and consequences for the analysis toolchain

_Decision_: :x: This proposal has been withdrawn.

#### Discussion/Enhancement: Support for vertical off-center writes
_Author:_ Rhea George (Vulcan), Oliver Fuhrer (Vulcan)

_Priority:_

_Description:_ some column based atmospheric science algorithms are intuitive to scientists to write in such a way that adjacent vertical layers are modified in conjunction. For example, if you take mass from the layer you are at and add it to the one below you, it is intuitive to write:
```python
with computation(BACKWARD), interval(1, None):
        if mass[0, 0, 0] > 0:
             delta = x * y
             mass[0, 0, 0]  = mass[0, 0, 0]  - delta
             mass[0, 0, -1] = mass[0, 0, -1] + delta 
```
Without off-center writes, we need to write it something like
```python
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
```python
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
Stencil version avoiding offcenter writes-introduce fields `upper_fix` and `lower_fix`:
```python
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

_Decision:_ off-center writes should only be allowed with the following restrictions:
- in the vertical direction
- in non parallel computations
- relative offsets (in any direction)

Open question: check if all cases can be converted to the current parallel model with workarounds


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
so it can be used with an empty `computation()`:
```python
with computation():  # Ok
    ...
```

_Decision:_ it should be removed in the future (maybe with a transition period where is deprecated)


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
 + [X] The specification order should match execution order when is a sequential computation, and can be any other order when is "parallel"



#### Discussion/Enhancement: Expose the iteration index (_positional computations_)
_Author:_ Till Ehrengruber (CSCS), Enrique G. P. (CSCS)

_Priority:_  

_Description:_ users may want to access the current iteration index in the computation and use it as a numerical value. Since a stencil program will normally contain several iteration loops with different bounds (as part of the extended computation domain), if GT4Py exposes the iteration index, it should be clearly defined how the exposed index maps to the computation domain, and possible what are the possible consequences/interactions with the merging of stages   ofstages merging step in the analysis phase.

_Proposals:_
The syntax could be something like the following approach, but it must be always clear to the user which iteration loop is being uexposed:
```python

with computation(), interval(...):
    field_a = field_b[-2, 0, 0] + field_b[1, 0, 0]
    
    I = index(I)
    field = field[idx, idx, idx]
    
    with axes(I, J, K) as i_index, j, k:  # option 2
        tmp = 3 * i_index * index(I)
        field_c = 3 * i_indextmp * (field_a[-1, -1, 0] - field_a[1, 1, 0])
        
    out = field_c[I-1]
    ...

    with axes(I, J, K) as i_index, j, k:  
        tmp = 3 * i_index
        field_c = 3 * i_indextmp * (field_a[-1, -1, 0] - 
```

Example: Cloud physics (TBD @tobias)

https://github.com/ofuhrer/HPC4WC/blob/55c9c9242ed6bc7810541725c2cfcf0dfa655aec/projects2020/group05/shalconv/kernels/stencils_part2.py#L101

https://github.com/ofuhrer/HPC4WC/blob/55c9c9242ed6bc7810541725c2cfcf0dfa655aec/projects2020/group05/shalconv/kernels/stencils_part2.py#L196


Should have the same restrictions as for loop variable (see discussion below).

_Decision:_ 
Syntax: use `index(AXES_NAME)` to get the actual iteration index  
Functionality: 
- only as a run time int value in an expression
- (Not as an index in field)

