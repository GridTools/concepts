# GTScript language design guideline

The following principles are a guideline for designing the GTScript DSL. We try to follow these principles if we can.
In some cases we cannot fulfill all principles and a trade-off has to be made and justified.

The principles are not magic, they mainly summarize the obvious.

Trivia: GTScript is an embedded DSL in Python, therefore language syntax is restricted to valid Python syntax.

1. **Language constructs should behave the same as their equivalent in other languages, especially as equivalent concepts
   in Python or well-known Python libraries (e.g. Numpy).**

   Motivation: The DSL should be readable by applying common sense and common programming language knowledge.

2. **Semantic differences should be reflected in syntactic differences.**

   Motivation: Spotting semantic differences is much harder than spotting syntactic differences.

3. **Regular use-cases should be simple, special cases can be complex.**

   Motivation: If a trade-off has to be made, the most common, standard use-cases should be expressed in the simplest
   possible way. To cover all cases, corner cases might require more complex language constructs.

4. **Language constructs are required to have an _unambiguous translation to parallel code_ and need to allow translation
   to efficient code _in the regular use-cases_.**

   Motivation: When translating DSL to executable code, we must not make correctness errors, therefore we cannot allow
   ambiguous language constructs.
   If we fail,

   - the user will run into hard to debug problems,
   - the toolchain developer cannot reason about the code and will fail in writing correct optimizations.

   On purpose, performance is second and, on purpose, the requirement to produce efficient code is restricted to regular use-cases. Obviously, for a performance portable language, the regular use-cases are required to have an
   efficient translation. But this principle acknowledges that we cannot exclude that for some special cases an
   efficient translation cannot be found.

# Parallel Model

The iteration domain is a 3d domain: `I` and `J` axes live on the horizontal spatial plane, and axis `K` represents the vertical spatial dimension.

A `gtscript.stencil` is composed of one or more `computation`. Each `computation` defines an iteration policy (`FORWARD`, `BACKWARD`, `PARALLEL`) and is itself composed of one or more non-overlapping vertical `interval` specifications, each one of them representing a vertical loop over with the iteration policy of the coputation. Each interval contains one or more statements.

The effect of the program is as if statements are executed as follows:

1. _computations_ are executed sequentially in the order they appear in the code,
2. vertical _intervals_ are executed sequentially in the order defined by the _iteration policy_ of the _computation_
3. every vertical _interval_ is executed as a sequential for-loop over the `K`-range following the order defined by the iteration policy,
4. for every _assignment_ inside the _interval_, first, the right hand side is evaluated in a parallel for-loop over the horizontal dimension(s), then, the resulting horizontal slice is assigned to the left hand side.
5. for `if`-`else` statements, the condition is evaluated first, then the `if` and `else` bodies are evaluated with the same rule as above.

### Example

In the following, the code snippets are not always complete GTScript snippets, instead parts are omitted (e.g. by `...`) to highlight
the important parts.

**Rule 4**

```python
with computation(...):
    with interval(...):
        a = b
```

translates to

```python
for k:
    tmp_b: IJField
    parfor ij:
        tmp_b = b
    parfor ij:
        a = tmp_b
```

which reflects principle 4, the translation to parallel code is unambigous.
Note: Removing the (in this case) unneeded temporary is up to optimization.

In the following examples, the translation of each right hand side to an intermediate temporary is implicit for
simplicity and to avoid distraction from the important aspects.

In the following, `start <= end`,

```python
with computation(FORWARD):  # Forward computation
    with interval(start, end):
        a = tmp[1, 1, 0]
        b = 2 * a[1, 1, 0]

with computation(BACKWARD):  # Backward computation
    with interval(start, -2):  # lower interval
        a = tmp[1, 1, 0]
        b = 2 * a[0, 0, 0]
    with interval(-2, end):    # upper interval
        a = 1.1
        b = 2.2

with computation(PARALLEL):  # Parallel computation
    with interval(start, end):
        a = tmp[1, 1, 0]
        b = 2 * a[0, 0, 0]
```

corresponds to the following pseudo-code:

```python
# Forward computation
for k in range(start, end):
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]
    parfor ij:
        b[i, j, k] = 2 * a[i+1, i+1, k]

# Backward computation
# upper interval
for k in reversed(range(end-2, end)):    
    parfor ij:
        a[i, j, k] = 1.1
    parfor ij:
        b[i, j, k] = 2.2
# lower interval
for k in reversed(range(start, end-2)): 
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]
    parfor ij:
        b[i, j, k] = 2 * a[i, j, k]

# Parallel computation
parfor k in range(start, end):
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]
    parfor ij:
        b[i, j, k] = 2 * a[i, j, k]
```

where `parfor` implies no guarantee on the order of execution.

<table><tr>
<td><details><summary>NumPy style</summary>
  
```python
# Domain definition
# i, I = domain_start_i, domain_end_i
# j, J = domain_start_j, domain_end_j

# Forward computation
for k in range(start, end):
    a[i:I, j:J, k] = tmp[i+1:I+1, j:J, k]
    b[i:I, j:J, k] = 2 * a[i+1:I+1, j+1:J+1, k]

# Backward computation
# upper interval
for k in reversed(range(end-2, end)):    
    a[i:I, j:J, k] = 1.1
    b[i:I, j:J, k] = 2.2
# lower interval
for k in reversed(range(start, end-2)):  
    a[i:I, j:J, k] = tmp[i+1:I+1, j+1:J+1, k]
    b[i:I, j:J, k] = 2 * a[i:I, j:J, k]

# Parallel computation
for k in random.shuffle(range(start, end)):
    a[i:I, j:J, k] = tmp[i+1:I+1, j+1:J+1, k]
    b[i:I, j:J, k] = 2 * a[i:I, j:J, k]
```

</details></td>
<td><details><summary>C++ style</summary>

```cpp
# Forward computation
for (int k=0; k < end; k++) {
    parallel_for (auto& i, j : ij_domain()) {
        a[i, j, k] = tmp[i+1, j+1, k]    
    }
    parallel_for (auto& i, j : ij_domain()) {
        b[i, j, k] = 2 * a[i+1, i+1, k]
    }
}

# Backward computation
# upper interval
for (int k=end-1; k >= end-2; k--) {
    parallel_for (auto& i, j : ij_domain()) {
        a[i, j, k] = 1.1
    }
    parallel_for (auto& i, j : ij_domain()) {
        b[i, j, k] = 2.2
    }
# lower interval
for (int k=end-2; k >= 0; k--) {
    parallel_for (auto& i, j : ij_domain()) {
        a[i, j, k] = tmp[i+1, j+1, k]
    }
    parallel_for (auto& i, j : ij_domain()) {
        b[i, j, k] = 2 * a[i, j, k]
    }

# Parallel computation
parallel_for (int k=0; k < end; k++) {
    parallel_for (auto& i, j : ij_domain()) {
        a[i, j, k] = tmp[i+1, j+1, k]
    }
    parallel_for (auto& i, j : ij_domain()) {
        b[i, j, k] = 2 * a[i, j, k]
    }
```
where `parallel_for` implies no guarantee on the order of execution.

</details></td>
</tr></table>



## Variable declarations

Variable declarations inside a computation are interpreted as temporary field declarations spanning the actual computation domain of the `computation` where they are defined.

### Example

```python
with computation(FORWARD):
    with interval(1, 3):
        tmp = 3
```

behaves like:

```python
tmp = Field(domain_shape)  # Uninitialized field (random data)
for k in range(0, 3):
    parfor ij:
        tmp[i, j, k] = 3   # Only this vertical range is properly initialized
```

Note: Demoting the temporary field to a local variable is up to optimization.

## Compute Domain

The computation domain of every statement is extended to ensure that any required data to execute all stencil statements on the compute domain is present.

### Example

On an applied example, this means:

```python
with computation(PARALLEL), interval(...):
    u = 1
    b = u[-2, 0, 0] + u[1, 0, 0] + u[0, -1, 0] + u[0, -2, 0]
```

translates into the following pseudo code:

```python
for k in range(start, end):
    parfor [i_start-2:i_end+1, j_start-1:j_end+2]:
        u[i, j, k] = 1
    parfor [i_start:i_end, j_start:j_end]:
        b[i, j, k] = u[i-2,j,k] + u[i+1,j,k] + u[i,j-1,k] + u[i,j-2,k]
```

## Conditionals

We distinguish between 2 kinds of conditionals (TODO thanks to proposal 4), `if`s on scalars, and `if`s on fields.

### `if` on scalars

Conditions on scalars are straight-forward, the condition is evaluated inside the vertical loop and normal rules apply for the statements in the branches.

**Example**

```python
with computation(...), interval(...):
    if scalar_flag:
        b = a
        c = b
    else:
        b = 1
        c = 1
```

translates to

```python
for k in range(start, end):
    if scalar_flag:
        parfor ij:
            b = a
        parfor ij:
            c = b
    else:
        parfor ij:
            b = 1
        parfor ij:
            c = 1

```

### `if` on fields

For conditions with fields, the `if`-`else`-statement behaves as if it is evaluated for each gridpoint without interference with other gridpoints. This is intuitive behavior, however hard to implement.

To comply with principle 4 we need an unambigous translation to parallel code and find an optimization
to efficient code.

**translation to parallel code**

For each IJ-plane:

- The condition is evaluated in a parfor loop and written to a _mask_ field.
- Each field that is referenced in either the `if` or `else` branch is versioned separately (i.e. if the field appears in both branches, each branch gets a version).
- All assignments inside the branches are done on versioned left hand sides.
- After the conditional, the field versions are written to their output counterparts taking into account the branch origin.

```python
with computation(...) with interval(...):
    if some_field > 0:
        out = in
    else:
        out = 2*in
```

translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask[i,j] = some_field[i,j,k] > 0

    parfor ij: # note that this is always just a copy and therefore it doesn't matter if in the same parfor ij
        in_if = in
        in_else = in
    parfor ij:
        if some_field_mask:
            out_if = in_if
    parfor ij:
        if not some_field_mask:
            out_else = 2 * in_else
    parfor ij:
        if some_field_mask:
            out = out_if
        if not some_field_mask:
            out = out_else
```

For nested conditionals

```python
with computation(...) with interval(...):
    if some_field > 0:
        if some_other_field < 0:
            out = some_field
        else:
            out = some_other_field:
    else:
        out = in
```

translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask = some_field > 0
    parfor ij:
        some_field_if_if = some_field
        some_other_field_if_else = some_other_field
        in_else = in
    parfor ij:
        if some_field_mask:
            if some_other_field_mask:
                out_if_if = some_field_if_if
    parfor ij:
        if some_field_mask:
            if not some_other_field_mask:
                out_if_else = some_other_field_if_else
    parfor ij:
        if not some_field_mask:
            out_else = in_else
    parfor ij:
        if some_field_mask:
            if some_other_field_mask:
                out = out_if_if
            else:
                out = out_if_else
        else:
            out = out_else
```

The previous examples don't illustrate the necessity of the parfor around each statement.
Here is "Eddie's example"

```python
with computation(...) with interval(...):
    if some_field > 0:
        tmp = inout
        inout = tmp[-1, 0, 0]
```

which translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask = some_field > 0
    parfor ij:
        inout_if = inout
    parfor i(-1,0)j
        if some_field_mask:
            tmp_if = inout_if
    parfor ij:
        if some_field_mask:
            inout_if = tmp_if # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask:
            inout = inout_if
```

# Translation to efficient code

Here we show how the examples given above can be made efficient by analysis and transformation.

### Simple "ternary-like" conditional

<details>

```python
with computation(...) with interval(...):
    if some_field > 0:
        out = in
    else:
        out = 2*in
```

which translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0

    parfor ij: # note that this is always just a copy and therefore it doesn't matter if in the same parfor ij
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
    parfor ij:
        if not some_field_mask:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]
```

```python
with computation(...) with interval(...):
    if some_field > 0:
        out = in
    else:
        out = 2*in
```

which translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0

    parfor ij: # note that this is always just a copy and therefore it doesn't matter if in the same parfor ij
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
    parfor ij:
        if not some_field_mask:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        if not some_field_mask:
            out[i, j, k] = out_else[i, j]
```

We can iteratively inline the values of temporaries, if they are never changed:

First iteration:

```python
for k in range(...):
    parfor ij:
        if some_field[i,j,k] > 0:
            out_if[i,j] = in[i,j, k]
    parfor ij:
        if not some_field[i,j,k] > 0:
            out_else[i,j] = 2 * in[i,j, k]
    parfor ij:
        if some_field[i,j,k] > 0:
            out[i,j,k] = out_if[i,j]
        if not some_field[i,j,k] > 0:
            out[i,j,k] = out_else[i,j]
```

Second iteration: note that we can assume that the condition does not change between :code:`parfor`'s, since :code:`some_field` doesn't change and the conditional expressions is the same.
 
```python
for k in range(...):
    parfor ij:
        if some_field[i,j,k] > 0:
            out[i,j,k] = in[i,j, k]
        if not some_field[i,j,k] > 0:
            out[i,j,k] = 2 * in[i,j, k]

```

finally, the two remaining branches of the condition are obviously the negation of each other and the condition does again not change, therefore it is valid to write

```python
for k in range(...):
    parfor ij:
        if some_field[i,j,k] > 0:
            out[i,j,k] = in[i,j, k]
        else:
            out[i,j,k] = 2 * in[i,j, k]

```

</details>

For this fairly basic example, we can also try that only fusing `parfor` and turning temporary fields into `const` variables.This is what the current pipeline does, and it might be enough to cover many common yet simple cases:

<details>

```python
with computation(...) with interval(...):
    if some_field > 0:
        out = in
    else:
        out = 2*in
```

which translates to

```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
    parfor ij:
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
    parfor ij:
        if not some_field_mask:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]
```
strategy: greedily fuse parfrors, top-to-bottom, if legal. then, turn unneeded temporaries into variables, and leave it to the compiler to get it right. (should be optimal in terms of non-register loads.)

1. fusion: legal since no race condition (`in` is only read)
```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
    parfor ij:
        if not some_field_mask:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]
```
2. fusion: legal although read adn write in in_f, since no offset.
```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
    parfor ij:
        if not some_field_mask:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]

3. fusion: legal although read and write in in_else, since no offset.
```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
        if not some_field_mask[i, j]:
            out_else[i, j] = 2 * in_else[i, j]
    parfor ij:
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]
4. fusion: legal although read adn write in out_if and out_else, since no offset.
```python
for k in range(...):
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
        in_if[i, j] = in[i, j, k]
        in_else[i, j] = in[i, j, k]
        if some_field_mask[i, j]:
            out_if[i, j] = in_if[i, j]
        if not some_field_mask[i, j]:
            out_else[i, j] = 2 * in_else[i, j]
        if some_field_mask:
            out[i, j, k] = out_if[i, j]
        else:
            out[i, j, k] = out_else[i, j]
```

all temporary fields are only used locally, and not read/written at offset, can therefore be scalars:
```python
for k in range(...):
    parfor ij:
        some_field_mask[0] = some_field[i, j, k] > 0
        in_if[0] = in[i, j, k]
        in_else[0] = in[i, j, k]
        if some_field_mask[0]:
            out_if[0] = in_if[0]
        if not some_field_mask[0]:
            out_else[0] = 2 * in_else[0]
        if some_field_mask[0]:
            out[i, j, k] = out_if[0]
        else:
            out[i, j, k] = out_else[0]
```
since all conditions are now unmodified scalars (could be declared `const`, that are defined in the same scope and only maybe negated, a compiler should be able to generate efficient code from this. 
</details>

### Nested conditional

<details>

For nested conditionals

```python
with computation(...) with interval(...):
    if some_field > 0:
        if some_other_field < 0:
            out = some_field
        else:
            out = some_other_field:
    else:
        out = in
```

translates to

```python
for k in range(...):
    #preparing outer conditional
    parfor ij:
        some_field_mask[i,j] = some_field[i,j,k] > 0
    parfor ij:
        some_field_if[i,j] = some_field[i,j,k] 
        some_other_field_if[i,j] = some_other_field[i,j,k]
        in_else = in
    #computations of if branch
    ##preparing inner conditional
    parfor ij:
        some_other_field_mask[i,j] = some_other_field_if[i,j,k] < 0
    parfor ij:
        some_field_if_if[i,j] = some_field_if[i,j]
        some_other_field_if_else[i,j] = some_other_field_if[i,j]
    ##computations of if-if branch
    parfor ij:
        if some_field_mask[i,j]:
            if some_other_field_mask[i,j]:
                out_if_if[i,j] = some_field_if_if[i,j]
    ##computations of if-else branch
    parfor ij:
        if some_field_mask[i,j]:
            if not some_other_field_mask[i,j]:
                out_if_else[i,j] = some_other_field_if_else[i,j]
    ##commiting writes of inner conditional
    parfor ij:
        if some_field_mask[i,j]:
            if some_other_field_mask[i,j]:
                out_if[i,j] = out_if_if[i,j]
            else:
                out_if[i,j]= out_if_else[i,j]
    #computations of else branch
    parfor ij:
        if not some_field_mask[i,j]:
            out_else[i,j] = in_else[i,j]
    #commiting writes of outer conditional
    parfor ij:
        if some_field_mask[i,j]:
            out[i,j,k] = out_if[i,j]
        else:
            out[i,j,k] = out_else[i,j]
```

First Iteration: inline some_field_mask, some_field_if, some_other_field_if, in_else

```python
for k in range(...):
    #computations of if branch
    ##preparing inner conditional
    parfor ij:
        some_other_field_mask[i,j] = some_other_field[i,j,k] < 0
    parfor ij:
        some_field_if_if[i,j] = some_field[i,j,k]
        some_other_field_if_else[i,j] = some_other_field[i,j,k]
    ##computations of if-if branch
    parfor ij:
        if some_field[i,j,k] > 0:
            if some_other_field_mask[i,j]:
                out_if_if[i,j] = some_field_if_if[i,j]
    ##computations of if-else branch
    parfor ij:
        if some_field[i,j,k] > 0:
            if not some_other_field_mask[i,j]:
                out_if_else[i,j] = some_other_field_if_else[i,j]
    ##commiting writes of inner conditional
    parfor ij:
        if some_field[i,j,k] > 0:
            if some_other_field_mask[i,j]:
                out_if[i,j] = out_if_if[i,j]
            else:
                out_if[i,j]= out_if_else[i,j]
    #computations of else branch
    parfor ij:
        if not some_field[i,j,k] > 0:
            out_else[i,j] = in_else[i,j]
    #commiting writes of outer conditional
    parfor ij:
        if some_field[i,j,k] > 0:
            out[i,j,k] = out_if[i,j]
        else:
            out[i,j,k] = out_else[i,j]
```

Second iteration: inline some_other_field_mask, some_field_if_if, some_other_field_if_else

```python
for k in range(...):
    #computations of if branch
    ##computations of if-if branch
    parfor ij:
        if some_field > 0:
            if some_other_field < 0:
                out_if_if = some_field
    ##computations of if-else branch
    parfor ij:
        if some_field > 0:
            if not some_other_field < 0:
                out_if_else = some_other_field
    ##commiting writes of inner conditional
    parfor ij:
        if some_field > 0:
            if some_other_field < 0:
                out_if = out_if_if
            else:
                out_if = out_if_else
    #computations of else branch
    parfor ij:
        if not some_field > 0:
            out_else = in
    #commiting writes of outer conditional
    parfor ij:
        if some_field > 0:
            out = out_if
        else:
            out = out_else
```

Third iteration: inline out_if_if, out_if_else

```python
for k in range(...):
    #computations of if branch
    ##commiting writes of inner conditional
    parfor ij:
        if some_field > 0:
            if some_other_field < 0:
                out_if = some_field
            else:
                out_if = some_other_field
    #computations of else branch
    parfor ij:
        if not some_field > 0:
            out_else = in
    #commiting writes of outer conditional
    parfor ij:
        if some_field > 0:
            out = out_if
        else:
            out = out_else
```

Fourth iteration: inline out_if, out_else

```python
for k in range(...):
    #commiting writes of outer conditional
    parfor ij:
        if some_field > 0:
            if some_other_field < 0:
                out = some_field
            else:
                out = some_other_field
        else:
            out = in
```
</details>

Also, when there are nested conditionals, but there are also other statements within the scope of the outer conditional, this still works:

<details>

For nested conditionals

```python
@gtscript.stencil
def stencil(in, out, some_field, some_other_field):
with computation(...) with interval(...):
    if some_field > 0:
        some_other_field *= -1;
        if some_other_field < 0:
            out = some_field
        else:
            out = some_other_field
    else:
        out = in
```

just renaming first:

```python
    some_field_mask = some_field > 0
    some_field_if = some_field
    some_other_field_if = some_other_field # no need for some_other_field_else since it is not written there
    out_if = out
    out_else = out
    in_else = in
    if some_field_mask:
        some_other_field_if *= -1;
        some_other_field_if_mask = some_other_field_if < 0
        some_other_field_if_if = some_other_field_if # these are 2D to 2D, not 3D to 2D as the outer ones
        some_field_if_if = some_field_if
        out_if_if = out_if
        out_if_else = out_if
        if some_other_field_if_mask:
            out_if_if = some_field_if_if
        else:
            out_if_else = some_other_field_if_if
        if some_other_field_if_mask:
            out_if = out_if_if
        else
            out_if = out_if_else
    else:
        out_else = in_else
        
    if some_field_mask:
       some_other_field = some_other_field_if
       
    if some_field_mask:
       out = out_if
    else:
       out = out_else
            
```

Now we can optimize as usual starting from the innermost scope.

1) `some_field` is not changed

```python
    some_field_mask = some_field > 0
    some_field_if = some_field
    some_other_field_if = some_other_field
    out_if = out
    out_else = out
    in_else = in
    if some_field_mask:
        some_other_field_if *= -1;
        some_other_field_if_mask = some_other_field_if < 0
        some_other_field_if_if = some_other_field_if # these are 2D to 2D, not 3D to 2D as the outer ones
        out_if_if = out_if
        out_if_else = out_if
        if some_other_field_if_mask:
            out_if_if = some_field_if
        else:
            out_if_else = some_other_field_if_if
        if some_other_field_if_mask:
            out_if = out_if_if
        else
            out_if = out_if_else
    else:
        out_else = in_else
    
    if some_field_mask:
       some_other_field = some_other_field_if
       
    if some_field_mask:
       out = out_if
    else:
       out = out_else
            
```

2) `some_other_field_if_if` is not modified either

```python
    some_field_mask = some_field > 0
    some_field_if = some_field
    some_other_field_if = some_other_field
    out_if = out
    out_else = out
    in_else = in
    if some_field_mask:
        some_other_field_if *= -1;
        some_other_field_if_mask = some_other_field_if < 0
        out_if_if = out_if
        out_if_else = out_if
        if some_other_field_if_mask:
            out_if_if = some_field_if
        else:
            out_if_else = some_other_field_if
        if some_other_field_if_mask:
            out_if = out_if_if
        else
            out_if = out_if_else
    else:
        out_else = in_else
    
    if some_field_mask:
       some_other_field = some_other_field_if
       
    if some_field_mask:
       out = out_if
    else:
       out = out_else
            
```

3) `some_field` is not changed, so inlining the mask is ok, and also `some_field_if` is not needed. Also `in` is not modified and can be inlined/

```python
    some_other_field_if = some_other_field
    out_if = out
    out_else = out
    if some_field > 0 :
        some_other_field_if *= -1;
        some_other_field_if_mask = some_other_field_if < 0
        out_if_if = out_if
        out_if_else = out_if
        if some_other_field_if_mask:
            out_if_if = some_field
        else:
            out_if_else = some_other_field_if
        if some_other_field_if_mask:
            out_if = out_if_if
        else
            out_if = out_if_else
    else:
        out_else = in
    
    if some_field > 0:
       some_other_field = some_other_field_if
       
    if some_field > 0 :
       out = out_if
    else:
       out = out_else
            
```

4) `out_if_if` and `out_if_else` are complementary, and `out_else` is complementary to those two

```python
    some_other_field_if = some_other_field
    out_if = out
    out_else = out
    if some_field > 0 :
        some_other_field_if *= -1;
        some_other_field_if_mask = some_other_field_if < 0
        if some_other_field_if_mask:
            out_if = some_field
        else:
            out_if = some_other_field_if
    else:
        out_else = in
    
    if some_field > 0:
       some_other_field = some_other_field_if
       
    if some_field > 0 :
       out = out_if
    else:
       out = out_else
            
```

5) `out_if` and `out_else` are complementary

```python
    some_other_field_if = some_other_field
    if some_field > 0 :
        some_other_field_if *= -1
        some_other_field_if_mask = some_other_field_if < 0
        if some_other_field_if_mask:
            out = some_field
        else:
            out = some_other_field_if
    else:
        out = in
    
    if some_field > 0:
       some_other_field = some_other_field_if            
```

5.1) If we did not have `some_other_field_if *= -1` same result as Linus!

```python
    if some_field > 0 :
        if some_other_field < 0:
            out = some_field
        else:
            out = some_other_field
    else:
        out = in
```
    
</details>

### Eddie's Example

<details>


```python
with computation(...) with interval(...):
    tmp = 0
    if some_field > 0:
        tmp = inout
        inout = tmp[-1, 0, 0]
```

which translates to

```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
    parfor ij:
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```

First iteration: inline temporary some_field_mask, tmp (but can't remove tmp write yet due to uninitialized remaining values)
```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
    parfor ij:
        inout_if = inout
        tmp_if = 0
    parfor ij:
        if some_field[i, j, k] > 0
            tmp_if = inout_if
    parfor ij:
        if some_field[i, j, k] > 0 
            inout_if[i, j, k] = tmp_if[i-1, j]
    parfor ij:
        if some_field[i, j, k] > 0
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```

remove shadowed write to tmp
```python
for k in range(...):
    parfor ij:
        inout_if = inout
    parfor ij:
        if some_field[i, j, k] > 0
            tmp_if = inout_if
        else:
            tmp_if = 0
    parfor ij:
        if some_field[i, j, k] > 0 
            inout_if[i, j, k] = tmp_if[i-1, j]
    parfor ij:
        if some_field[i, j, k] > 0
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
        else:
            tmp[i, j, k] = 0
```

Inline the first read of inout_if, since all later reads are shadowed.

```python
for k in range(...):
    parfor ij:
        if some_field[i, j, k] > 0
            tmp_if = inout
        else:
            tmp_if = 0
    parfor ij:
        if some_field[i, j, k] > 0 
            inout_if[i, j, k] = tmp_if[i-1, j]
    parfor ij:
        if some_field[i, j, k] > 0
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
        else:
            tmp[i, j, k] = 0
```
inline tmp_if (can not fully inline it since otherwise inout conflicts in final parfor, therefore also hafe to keep the initial write.)
```python
for k in range(...):
    parfor ij:
        if some_field[i, j, k] > 0
            tmp_if = inout
        else:
            tmp_if = 0
    parfor ij:
        if some_field[i, j, k] > 0 
            if some_field[i-1, j, k] > 0:
                inout_if[i, j, k] = inout[i-1, j]
            else:
                inout_if[i, j, k] = 0
    parfor ij:
        if some_field[i, j, k] > 0
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
        else:
            tmp[i, j, k] = 0
```

This one's a bit tricky: we can remove the first condition, since
    (1)the else-case is never read and
    (2)the if-case is always protected by the same condition, BUT shifted accordingly
    
```python
for k in range(...):
    parfor ij:
        tmp_if = inout
        
    parfor ij:
        if some_field[i, j, k] > 0 
            if some_field[i-1, j, k] > 0:
                inout_if[i, j, k] = tmp_if[i-1, j]
            else:
                inout_if[i, j, k] = 0
                
    parfor ij:
        if some_field[i, j, k] > 0
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
        else:
            tmp[i, j, k] = 0
```


inline inout_if: it is written under the same conditions that it is read later.

```python
for k in range(...):
    parfor ij:
        tmp_if = inout
    parfor ij:
        if some_field[i, j, k] > 0
            if some_field[i-1, j, k] > 0:
                inout[i, j, k] = tmp_if[i-1, j]
            else:
                inout[i, j, k] = 0
            tmp[i, j, k] = tmp_if[i, j]
        else:
            tmp[i, j, k] = 0
```


If we assume that tmp is not API, we can remove the final write since it is not used after:
```python
for k in range(...):
    parfor ij:
        tmp_if = inout
    parfor ij:
        if some_field[i, j, k] > 0 
            if some_field[i-1, j, k] > 0 
                inout[i, j, k] = tmp_if[i-1, j]
            else:
                inout[i, j, k] = 0
```
</details>

Here we had assumed that our pipeline will be able to identify when two conditional predicates are the same (or complimentary). It is not conclusively clear yet if this will always be possible, although we can consider this given for the common cases. Here is an attempt that does not make this assumption, initially starting with a similar greedy fuse-and-demote approach as the current pipeline does:

<details>

```python
with computation(...) with interval(...):
    tmp = 0
    if some_field > 0:
        tmp = inout
        inout = tmp[-1, 0, 0]
```

which translates to

```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
    parfor ij:
        some_field_mask[i, j] = some_field[i, j, k] > 0
    parfor ij:
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```
1. fusion: trivial
```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
        some_field_mask[i, j] = some_field[i, j, k] > 0
    parfor ij:
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```
2. fusion: tmp no offset.
```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
        some_field_mask[i, j] = some_field[i, j, k] > 0
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
    parfor ij:
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```
3. fusion: inout_if  no offset, tmp_if only overwrites..
```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
        some_field_mask[i, j] = some_field[i, j, k] > 0
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
    parfor ij:
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```
the next fusion will skip the first parfor, since it would lead to a conflict in tmp_if.
4. fusion: the second parfors can be fused though, since tmp_if is not modified, all others have no offset.
```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
        some_field_mask[i, j] = some_field[i, j, k] > 0
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```
now, `some_field_mask`, `tmp_if` and `inout_if` can't be variables since they are communicated from the first parfor to the second. 

```python
for k in range(...):
    parfor ij:
        tmp[i, j, k] = 0
        some_field_mask[i, j] = some_field[i, j, k] > 0
        inout_if[i, j] = inout[i, j, k]
        tmp_if[i, j] = tmp[i, j, k]
        if some_field_mask[i, j]:
            tmp_if[i, j] = inout_if[i, j]
    parfor ij:
        if some_field_mask[i, j]:
            inout_if[i, j] = tmp_if[i-1, j] # TODO here is a version name mistake if it should be a general algorithm
        if some_field_mask[i, j]:
            inout[i, j, k] = inout_if[i, j]
            tmp[i, j, k] = tmp_if[i, j]
```

finally, inlining the non-conditional cases:
```python
for k in range(...):
    parfor ij:
        if some_field[i, j, k] > 0:
            tmp_if[i, j] = inout[i, j, k]
        else:    
            tmp_if[i, j] = 0
    parfor ij:
        if some_field[i, j, k] > 0:
            if some_field[i, j, k] > 0:
                inout[i, j, k] = tmp_if[i-1, j]
```
</details>

Compared to the first approach, this reads `some_field` in the first parfor, but only reads `some_field` at 1 offset in the second `parfor`, while the first approach had read it only in the second parfor, but at two offsets. Depending on hardware and further transformations, these approaches may or may not result in different performance.

### Used Trafos:

A start at summarizing the transformations that did most of the work above:

1) Inline temporary
    
    Matches:
    * A write statement to a temporary value. it can be split into multiple writes by(inner) conditions, where the write occurs in all branches.
    * the write can further be inside an (outer) condition
    * A read statement after such a write statement, without an intermediate write to any gridpoint,
    * if there is an outer condition around the write, the read must be protected under the same condition, meaning the same statement and the involved values haven't changed in between.
    * The target of the read statement does not appear with an offset in the value of the write statement. (otherwise race condition)
    
    Actions:
    
    1. replace all reads of the temporary by the value found in the write statement, possibly inserting  conditionals. if the read is at a different offset than the write, also shift the condition by the same offset. 
    3. if all reads have been removed, or the entire field is guaranteed to be overwritten before the next read, remove write as well

2) remove writes that are shadowed by a condition

    Matches:
    * two writes to same field, where the later modifies a subset of gridpoints that are written to by the first, due to a condition around the second
    * the value is not read in between the two writes.

    Actions:
    
    1. remove the first write, and move it to the else (or if) branch of the second.
