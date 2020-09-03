The iteration domain is a 3d domain: `I` and `J` axes live on the horizontal spatial plane, and axis `K` represents the vertical spatial dimension.

A `gtscript.stencil` is composed of one or more `computation`. Each `computation` defines an interation policy (`FORWARD`, `BACKWARD`, `PARALLEL`) and is itself composed of one or more non-overlapping vertical `interval` specifications, each one of them representing a vertical loop over with the iteration policy of the coputation. Each interval contains one or more statements.

The effect of the program is as if statements are executed as follows:

- *computations* are executed sequentially in the order they appear in the code,
- vertical *intervals* are executed sequentially in the order defined by the *iteration policy* of the *computation*
- every vertical *interval* is executed as a sequential for-loop over the `K`-range following the order defined by the iteration policy,
- every *assignment* inside the *interval* is executed as an immediate assigment of all field values from the r.h.s. to the l.h.s. (implementation detail: fields that appear on the r.h.s. and the l.h.s. are versioned)

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
        
with computation(...):
    with interval(start, end):
        a = a[-1,0,0] # self assignment with shift
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

parfor k in range(start, end):
    parfor ij:
        tmp_a[i, j, k] = a[i, j, k]
    parfor ij:
        a[i, j, k] = tmp_a[i-1, j, k]
```

where `parfor` implies no guarantee on the order of execution.


### Variable declarations

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

### Compute Domain

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
