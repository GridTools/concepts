The iteration domain is a 3d domain: `I` and `J` axes live on the horizontal spatial plane, and axis `K` represents the vertical spatial dimension.

A `gtscript.stencil` is composed of one or more `computation`. Each `computation` defines an interation policy (`FORWARD`, `BACKWARD`, `PARALLEL`) and is itself composed of one or more non-overlapping vertical `interval` specifications, each one of them representing a vertical loop over with the iteration policy of the coputation. Each interval contains one or more statements.

The effect of the program is as if statements are executed as follows:

- *computations* are executed sequentially in the order they appear in the code,
- vertical *intervals* are executed sequentially in the order defined by the *iteration policy* of the *computation*
- every vertical *interval* is executed as a sequential for-loop over the `K`-range following the order defined by the iteration policy,
- every *statement* inside the *interval* is executed as a parallel for-loop over the horizontal dimension(s) with no guarantee on the order.

#### Example
On an applied example (by definition `start <= end`):

```python
with computation(FORWARD):  # Forward computation
    with interval(start, end):
        a = tmp[1, 1, 0]
        b = 2 * a[1, 1, 0]

with computation(BACKWARD):  # Backward computation
    with interval(start, -2):  # interval A
        a = tmp[1, 1, 0]
        b = 2 * a[0, 0, 0]
    with interval(-2, end):    # interval B
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
for k in reversed(range(end-2, end)):    # interval B
    parfor ij:
        a[i, j, k] = 1.1
    parfor ij:
        b[i, j, k] = 2.2
for k in reversed(range(start, end-2)):  # interval A
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


## Variable declarations

Variable declarations inside a computation are interpreted as temporary field declarations spanning the actual computation domain of the `computation` where they are defined.

#### Example
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

#### Example
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

GTScript supports 3 kinds of conditionals:

- compile-time conditionals using `with CONDITION()` (former `if __INLINED`)
- run-time conditionals on scalar expressions using `if` statements 
- run-time conditionals on field expressions (meaning a condition on the field value at the gridpoint of the current iteration or at a local offset of it) using ternary operators

### Compile-time conditionals (`with CONDITION()`)

This kind of conditional works like a conditional preprocessor directive which activates/deactivates certain parts of the GTScript program based on values known at compile-time. It might be used inside a `computation` (at any level) or even outside:

```python
with computation():
    with interval(...):
        a *= 2.5

with CONDITION(SHIFT_FLAG):
    with computation():
        with interval(start, end):
            b = a[1,1,0]
            with CONDITION(ADD_A_FLAG):
                a = a + 1
            with CONDITION(not ADD_A_FLAG):
                a = a - 1
```

### Run-time conditionals on scalar expressions (`if` statement)

- Conditions in `if` statements are evaluated at run-time and are required to be of scalar type, e.g. scalar variables (not fields) or scalar expressions (no fields involved).
- Each statement inside the if and else branches is executed as a `parfor` loop (same as statements outside of branches).

Rationale:
- If the condition is on a field, it's extremely complex to catch all corner cases if the parfor-loop is applied on each statement. E.g. think about a case where the `if`-branch updates a field for some points which is read in the `else`-branch (requires buffering of the original input).
- If the parfor would be applied outside of the `if`, the model would contain some unexpected limits:
    1. Removing an `if True:` will change behavior.
    2. Changing an `if __INLINE(flag)` to a runtime `if flag:` will change behavior.

#### Example

```python
with computation():
    with interval(...):
        if my_config_var:
            a = 1
            b = 2
        else:
            a = 2
            b = 1
```

translates to: 

```python
for k in range(start, end):
    if my_config_var:
        parfor ij:
            a[i, j, k] = 1
        parfor ij:
            b[i, j, k] = 2
    else:
        parfor ij:
            a[i, j, k] = 2
        parfor ij:
            b[i, j, k] = 1
```

A more complicated case with extended compute domain:

```python
with computation():
    with interval(...):
        if my_config_var:
            tmp = in_field[-1,0,0]
            tmp2 = tmp[-1,0,0]
        else:
            tmp2 = 1
        out_field = tmp2[-1,0,0]
```

translates to: 

```python
for k in range(start, end):
    if my_config_var:
        parfor [i_start-2:i_end, j_start:j_end]:
            tmp[i, j, k] = in_field[-1,0,0]
        parfor [i_start-1:i_end, j_start:j_end]:
            tmp2[i, j, k] = tmp[-1,0,0]
    else:
        parfor [i_start-1:i_end, j_start:j_end]:
            tmp2[i, j, k] = 1
    parfor ij:
        out_field[i, j, k] = tmp2[-1,0,0]
```

The following case is illegal:

```python
with computation():
    with interval(...):
        tmp: Field[[I, J, K], float] = 1
        if tmp == 1: # error, tmp is a temporary field
            a = 1
```
but it would be legal when `tmp` is a scalar value:
```python
with computation():
    with interval(...):
        tmp: int = 1
        if tmp == 1: # ok, tmp is a scalar
            a = 1
```

### Ternary expressions

- Conditions in ternary expressions can be of field type.

#### Examples

```python
with computation():
    with interval(...):
        out_field = in_field if in_field > 0 else -in_field
```

for more complicates expressions inside the branches, use a function.
