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
```

## Conditionals

GTScript supports 2 kinds of conditionals:

- conditionals on scalar expressions or compile-time constants 
- conditionals on field expressions (meaning a condition on the field value at the gridpoint of the current iteration or at a local offset of it)

### Conditionals on scalar expressions or compile-time constants

- Conditions in `if` statements are evaluated at run-time and are required to be of scalar type, e.g. scalar variables (not fields) or scalar expressions (no fields involved).
- Each statement inside the if and else branches is executed as a `parfor` loop (same as statements outside of branches).
- There is no restriction on the body of the statement.
- For consistency sake with the following we use the condition inside the loop as a concept (the implementation detail to not do this should not matter here)

Rationale:
- Adding an `if True:` does not change the behavior at all.
- The way the programmer thinks about execution is the same if within an if or not.
- This makes function calls and scoped code behave the same.

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
    parfor ij:
        if my_config_var:
            a[i, j, k] = 1
    parfor ij:
        if my_config_var:
            b[i, j, k] = 2
    parfor ij:
        if not my_config_var:
            a[i, j, k] = 2
    parfor ij:
        if not my_config_var:
            b[i, j, k] = 1
```

### Conditionals on field expressions

- Conditions in `if` statements are evaluated at run-time and are of storage type, e.g. fields or temporary variables.
- The condition is evaluated for all gridpoints
- Each statement inside the if and else branches is executed as a `parfor` loop (same as statements outside of branches).
- Inside the if and else blocks the same field cannot be written to **and** read with an offset (order does not matter).
- The evaluated condition is read to decide which branch of the if-else statement is traversed inside the loop.

Rationale:
- If the offset read-write is disallowed the user can reason about the execution.
- There is no real use-case that requires off center read and writes
- If a problem pops up, creating a second if-statement can resolve this quite easily
- We do not want to do 3 automatically since the user needs to think about what he wants.


#### Example

```python
with computation():
    with interval(...):
        if my_field_var:
            a = 1
            b = 2
        else:
            a = 2
            b = 1
```

translates to: 

```python
for k in range(start, end):
    parfor ij:
        condition[i,j,k] = my_field_var
    parfor ij:
        if condition[i,j,k]:
            a[i, j, k] = 1
    parfor ij:
        if condition[i,j,k]:
            b[i, j, k] = 2
    parfor ij:
        if not condition[i,j,k]:
            a[i, j, k] = 2
    parfor ij:
        if not condition[i,j,k]:
            b[i, j, k] = 1
```

The following case is illegal:

```python
with computation():
    with interval(...):
        tmp: Field[[I, J, K], float] = 1
        if tmp == 1: # tmp is a temporary field
            a = 1
            b = a[1, 0, 0] # we read and write a
```

### Discarded Idea

Just here for completeness sake and not formal at all.
Initially the idea was that if there is no else block, we can have unrestricted field accesses inside the if for additional functionality without complicating the thought process as the order is clear.

So 

```python
if c[0, 0, 0]:
    a = 1
    b = a[1, 0, 0]
```
would be legal code.

Discussing this internally in the team it was unclear what the result of this should be since the default behaviour (no if) is extending the stage `a = 1` would not directly apply and there could be an interpretation following this:

```python

# input variables
a = [ 10, 10, 10, 10, 10]
b = [ 42, 42, 42, 42, 42]
c = [ False, True, False, False, False]

# computational domain [ Halo, Compute, Compute, Compute, Halo]
domain = (3)
origin = (1)

# Stencil
with computation(PARALLEL), interval(...):
  if c[0, 0, 0]:
    a = 1
    b = a[1, 0, 0]

# output possibility 1 
a = [10, 1, 10, 10, 10]
b = [42, 10, 42, 42, 42]
# output possibility 2
a = [10, 1, 1, 10, 10]
b = [42, 1, 42, 42, 42]
```

and since there was too much internal dispute we decides to discard this for now. 

The two possible translations would be either:
```python
for k in domain[2]:
  for i in (0, domain[0]+1):
    if c[i, j, k]:
      a[i, j, k] = 1    
  for i in (1, domain[0]):
    if c[i, j, k]:
      b[i, j, k] = a[i+1, j, k]
```
or
```python
for k in domain[2]:
  for i in (0, domain[0]+1) and (c[i] or c[i-1]):
    a[i, j, k] = 1    
  for i in (1, domain[0]) and c[i]:
    b[i, j, k] = a[i+1, j, k]
```