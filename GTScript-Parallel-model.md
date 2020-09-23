# GTScript language design guideline

The following principles are a guideline for designing the GTScript DSL. We try to follow these principles if we can.
In some cases we cannot fulfill all principles and a trade-off has to be made and justified.

The principles are not magic, they mainly summarize the obvious.

Trivia: GTScript is an embedded DSL in Python, therefore language syntax is restricted to valid Python syntax.

1. **Language constructs should behave the same as their equivalent in other languages, especially as equivalent concepts
   in Python or well-known Python libraries (e.g. NumPy).**

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
