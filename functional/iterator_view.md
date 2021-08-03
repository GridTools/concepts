# Yet Another New Model for Unstructured 

## Definitions

- **Domain** - a set of points

- **Field** - a function from point to some value

- **Offset(s) Literal** - specifies the position of one point against another. The formulation is very generic on purpose. The believe is that the way how offsets are defined is orthogonal to the execution model. The important part that it should be *literal*. I.e. offsets should be known in generation/compile time. 

- **Iterator** - represents the position within the field. You can do two things with the iterator:
  - apply offsets (aka **shift**);
  - dereference (aka **deref**).

  The `shift` should have the following feature: for every iterator `i` and offsets literals `A` and `B`, there is a literal `C`, such that:
  ```
   shift(C)(i) == shift(B)(shift(A)(i))
  ```
  In other way -- shifts are foldable. Any composition of shifts can be presented as a single shift.
  
  The *deref* returns not the value directly but the optional. Deref returns **None** if the value was not set in the current position. 

- **Stencil** - a function that takes iterators and returns a value (or a tuple of values). The simplest possible stencil is *deref*.

- **Fencil** - a list of stencil closures (closure is a stencil, input and output fields and the domain).


## Lifting

Stencils are not composable -- they take iterators but returns values. Let us try to fix that introducing **lift** builtin that takes a stencil and return a function from iterators to iterator(s). Let us call the lift result **Lifted Stencil**. Semantics: derefencing the shifted iterator(s) returned by the lifted stencil is same as the result of the stencil with shifted arguments (with the same Offset(s) literal).

 
## Paralellism and polymorphism

We can execute a fencil by applying the stencil to each point of the domain in parallel if there are no vertical dependencies. This is the maximum possible parallelism.

If there are vertical dependencies, stencils can be defined on the whole columns instead of individual points. In this case fencil executes the stencil for each column in the domain in parallel.

To achieve the composability we want to compose value and column stencils. It is easy to do if we assume that each value stencil can take columns as well. In this case it executes the column pointwise.

## Easy builtins

Arithmetic operations and conditionals are the bare minimum set of builtins needed for point stencils. Let us focus on summation builtin. The most straitforward and conceptually simple is to define it like that:
```python 
def sum(a, b): return a + b  
```
This does not work because we need point/column polimorhism. The working implementation should look like:
```python
def sum(a, b):
  return (column(a[x] + b[x] for x in a.indices)
    if are_columns(a, b) else a + b)
```

Alternatively we can combine summation with the *deref* and the *lift* in the same builtin:
```python
@lift
def mega_sum(a, b):
   return sum(deref(a), deref(b))
```

If all the builtins are defined that way, we can get rid of using `lift` and `deref` in the stencil composition. That is probably a good idea for the frontend, but not for the intermediate level. Here the illustration. Suppose we have a stencil:
```python
def foo(a, b, c): return hannes_sum(hannes_sum(a, b), c)
```
comparing to
```python
def foo(a, b, c): return sum(sum(deref(a), deref(b)), deref(c))
```
And we are lowering this stencil to C++ in the context of parallel
execution (pointwise). No lifting is needed in this case and C++ code should look something like (suppose we have iterator-like args in C++ as well):
```C++
auto foo(auto a, auto b, auto c) { return (*a + *b) + *c; }
```
In `sum` case there is no explicit or implicit  lifting. the lowering can be implemented without addional transformation.

In `mega_sum` case, in between inner and outer `mega_sum` there is an implicit `lift` that is followed by immediate `deref`. Naive lowering of that construction will be like that (suppose we implement lowering by inlining):

```C++
auto foo(auto a, auto b, auto c) {
   auto tmp = make_inline(
     [](auto a, auto b) { return *a + *b; },
     a, b);
   return *tmp + *c;
}
```
Which is correct but pessimized way to go. To avoid this pessimization we need to perform some AST transformation. 

## Scanning

TODO(anstaf): scanning is the same thing as for cartesian.

* **scan**: a function with the signature:
  ```python
  scan(scan_pass: ScanPass, is_forward: bool, init) -> Stencil
  ```
  It returns a stencil that executes a vertical loop. `scan_pass` serves as a body of the loop; `is_forward` defines direction. The values that `scan_pass` returns are written to the returning column and also passed to the next invocation of `scan_pass` as a first parameter. On the first invocation `init` value is passed to `scan_pass`. Note that the scan returns a stencil that can be applied only to column iterators, but not to point iterators.


## Reduce and sparse fields

Say we have the data that is defined on unstructed mesh. And we are applying a stencil on cells over some domain. The stencil takes a two arguments and sums all multiplications of values in the neighbor edges:

```python
def val(v):
  return 0 if v is None else v

# maximum number of edges is 4
def foo(a, b):
  return (val(deref(shift(c2e(0))(a))) * val(deref(shift(c2e(0))(b)))
    + val(deref(shift(c2e(1))(a))) * val(deref(shift(c2e(1))(b)))
    + val(deref(shift(c2e(2))(a))) * val(deref(shift(c2e(2))(b)))
    + val(deref(shift(c2e(3))(a))) * val(deref(shift(c2e(3))(b)))
    )
```

Let as split the offset like `c2e(i)` into `c2e` and `nth(i)`. No we can now rewrite `foo`:

```python
def foo(a, b):
  a = shift(c2e)(a)
  b = shift(c2e)(b)
  return (val(deref(shift(nth(0))(a))) * val(deref(nth(edge(0))(b)))
    + val(deref(shift(nth(1))(a))) * val(deref(shift(nth(1))(b)))
    + val(deref(shift(nth(2))(a))) * val(deref(shift(nth(2))(b)))
    + val(deref(shift(nth(3))(a))) * val(deref(shift(nth(3))(b)))
    )
```

To support that pattern we introduce a builtin **reduce**. With the signature is:


```python
def reduce(fun, init) -> Stencil
```

Now we can rewrite `foo` again:

```python
def foo(a, b):
    return reduce(lambda acc, a, b: return acc + a * b, 0)(shift(c2e)(a), shift(c2e)(b))
    
```

The implementation of `reduce` could look like: 

```python
def reduce(fun, init):
    def sten(*args):
        assert check_that_all_args_are_comatible(*args)
        first_arg = args[0] 
        n = get_maximum_number_of_neigbors_from_args(first_arg)
        res = init
        for i in range(n):
            # we can check a single argument
            # because all arguments share the same pattern
            if deref(shift(nth(i))(first_arg)) is None:
                break
            res = fun(res, *(deref(shift(nth(i))(a)) for a in args))
        return res
    return sten
```

Let us call *Iterator* that can be shifted by `nth(i)` **SparseField**.
Two *SparseField*s are compatible if they are constructed with the *shift* with the same *offsets literal*. In this case they have the same maximum number of neighbors.


## Building a neighbor table

For each argument of each stencil closure within the fencil we can build its own neighbor table.
The trick here is to collect all the `shift`'s and extract offsets literals from them.

## Converting from unstructured offsets to strided offsets with colored fields

## IR Grammar in EBNF

```ebnf=
program = { function_definition | fencil_definition | setq };
function_definition =
    "(", "defun", id, "(", {id}, ")", expr, ")";
fencil_definition = "(", "defen", id, "(", {id}, ")",
    { "(", 
        ":domain", expr ,
        ":stencil", expr,
        ":outputs", {id},
        ":inputs", {id},
    ")" },
")";
setq = "(", "setq", id, expr, ")";
expr =
    int_literal    |
    float_literal  |
    offset_literal |
    id             |
    lambda         |
    fun_call       ;
lambda = "(", "lambda", "(", {id}, ")", expr, ")";
fun_call = "(", expr, {expr}, ")";
```

`let` is optional and can be added to expressions.

```ebnf
let = "(", "let", "(", {"(", id, expr, ")"}, ")", expr, ")";
```

```lisp
(let ((a <a_expr>) (b <b_expr>)) <expr>)
```

is the same as 

```lisp
(lambda (a b) <expr>)(<a_expr> <b_expr>)
```

```lisp=
;; minimalistic example of the IR script
(defen my_copy_fencil(x y z in out)
  (:domain (cartesian 0 x 0 y 0 z)
   :stencil deref
   :output out
   :input in
  ))
```

## Builtins


### Concepts

- **Domain** - can be followed by the `:domain` designator in the fencil definition. 
- **Value** - a type that potentially can be stored in the field.
- **Location** - one of `:cell`, `:vertex` or `:edge` 
- **Offset** - a designator that can passed to the `shift` call; the set of cartesian offsets could be `:i`, `:j`, `:k` and `:1`, `:2`, ...; for unstructured we could introduce smth like `:e2v`, `:v2e` ...;
- **Field<Value>** - can be followed by `:output` or `:input` in the fencil definition.
- **Column<Value>** - represents a column within the field; the height of the column is determined from *Domain* which is expected to be in the context. *Column<Value>* models *Value* concept.
- **Optional<Value>** - Value or `None`.
- **Tuple<...>** - just a tuple.
- **Iterator<Value>** - supports `shift` and `deref`
- **Stencil<R, Args...>** - a function with the signature: `R(Iterator<Args>...)`


### Signatures

```cpp=
  Domain cartesian(int i0, int i1, int j0, int j1, int k0, int k1);
  Domain unstructured(Location, int hor0, int hor1, int k0, int k1);

  // Iterators API
  Stencil<Iterator<V>, V> shift(Offsets...);
  Stencil<Optional<V>, V> deref;

  Stencil<Iterator<R>, Args...> lift(Stencil<R, Args...>);
  Tuple<Stencil<Iterator<Rs>, Args...>...> lift(Stencil<Tuple<Rs...>, Args...>);
  
  Stencil<Column<V>, Column<Args>...> scan(
    bool is_forward, V init, V pass(V, Iterator<Args>...));
    
  Stencil<V, Args...> reduce(V init, V fun(V, Args...));
  
  // Optional API
  V opt_deref(Optional<V>);
  bool is_none(Optional<V>);
  
  // Tuple API
  Tuple<Vs...> make_tuple(Vs...);
  auto nth(int, Tuple<Vs...>);
  
  // Those helpers are needed
  // if we do not want to support implicit type conversions
  Column<V> promote(V);
  To cast<To>(From);
  
  // ternary operator
  V if(bool, V, V);
  Column<V> if(Column<bool>, Column<V>, Column<V>);
  
  // conditionals 
  bool eq(V, V);
  Column<bool> eq(Column<V>, Column<V>);
  
  bool less(V, V);
  Column<bool> less(Column<V>, Column<V>);
  
  // logical
  bool not(bool);
  Column<bool> not(Column<bool>);
  
  bool and(bool, bool);
  Column<bool> and(Column<bool>, Column<bool>);

  bool or(bool, bool);
  Column<bool> or(Column<bool>, Column<bool>);
  
  // arithmetic
  V negate(V);
  Column<V> negate(Column<V>);
  
  V plus(V, V);
  Column<V> plus(Column<V>, Column<V>);
  
  V minus(V, V);
  Column<V> minus(Column<V>, Column<V>);
  
  V mult(V, V);
  Column<V> mult(Column<V>, Column<V>);
  
  V divides(V, V);
  Column<V> divides(Column<V>, Column<V>);
```
