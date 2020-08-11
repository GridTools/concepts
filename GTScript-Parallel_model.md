## GTScript Parallel Model

The iteration domain is a 3d domain: `I` and `J` axes live on the horizontal spatial plane, and axis `K` represents the vertical spatial dimension.

A GTScript `computation` is composed by one or more vertical `interval` specifications, each one of them representing a vertical loop over its vertical iteration range with a given iteration policy (`FORWARD`, `BACKWARD`, `PARALLEL`). Vertical `interval`s defined inside a computation should not have overlapping iteration ranges.

Every statement inside a vertical interval is executed on the full horizontal `IJ` plane before the next one is executed. Individual statements of a computation are considered embarrassingly parallel in the horizontal and therefore there are no guarantees about the loop order in the `IJ` plane. Every iteration of the `K`-loop defined by a vertical interval can only start after the previous iteration have finished. Vertical intervals defined in a `computation` will be sorted according to the iteration policy and then executed in order, that is, every vertical interval will start its execution after all the previous vertical intervals in the iteration order have been fully executed. In the case of a `PARALLEL` iteration policy, vertical intervals will be ordered in any implementation-specific way (no guarantees).

To summarize the model, we can say that:

- *computations* are executed sequentially in the order they appear in the code,
- vertical *intervals* are executed sequentially in the order defined by the *iteration policy* of the  *computation* (where `PARALLEL` means that order is not relevant and thus it will be implementation-specific),
- *statements* inside *intervals* are executed as (sequential) for-loops 
  over the `K`-range following the order defined by the iteration policy,
- a *statement* inside the *interval* is executed as a parallel for-loop over the horizontal dimension(s) with no guarantee on how statements are executed.

#### Example
On an applied example, this means (by definition `start <= end`):

```python
with computation(FORWARD):
    with interval(start, end):
        a = tmp[1, 1, 0]        # statement1
        b = 2 * a[1, 1, 0]      # statement2

with computation(BACKWARD):
    with interval(start, end):
        a = tmp[1, 1, 0]        # statement3
        b = 2 * a[0, 0, 0]      # statement4
with computation(PARALLEL):
    with interval(start, end):
        a = tmp[1, 1, 0]        # statement5
        b = 2 * a[0, 0, 0]      # statement6
```

corresponds to the following pseudo-code:

```python
for k in range(start, end):
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]    # statement1
    parfor ij:
        b[i, j, k] = 2 * a[i+1, i+1, k]  # statement2

for k in reversed(range(start, end)):
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]    # statement3
    parfor ij:
        b[i, j, k] = 2 * a[i, j, k]      # statement4
 
parfor k in range(start, end):
    parfor ij:
        a[i, j, k] = tmp[i+1, j+1, k]    # statement5
    parfor ij:
        b[i, j, k] = 2 * a[i, j, k]      # statement6
```

where `parfor` means that there is no guarantee of the order in which the iteration is performed. Additionally, the following restrictions apply:

- No self-assignments if `i != 0` or `j != 0`


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
tmp = Field(domain_shape)  # it contains random data at this point
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
