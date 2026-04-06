# Functional Patterns

Sources: Production functional codebases, FP community conventions, gamla library patterns

Covers: signal-to-noise, flat composition, point-free style, eta reduction,
pipeline construction, purity, immutability, currying, avoiding ambiguity.

## Signal-to-Noise Optimization

Duplication is easy to spot by the noisy feeling you get reading code.
When a pattern repeats with minor variation, there is almost always a
higher-level abstraction that eliminates the repetition.

### Recognizing Noise

Avoid:
```python
f = gamla.remove(
    gamla.anyjuxt(
        gamla.equals("hello"),
        gamla.equals("hi"),
        gamla.equals("what's up"),
        gamla.equals("good afternoon"),
        gamla.equals("morning"),
        gamla.equals("good evening"),
    )
)
```

The repetition of `gamla.equals` is noise. The reader sees the pattern
immediately but must parse it N times.

Prefer:
```python
f = gamla.remove(
    gamla.contains(
        {
            "hello",
            "hi",
            "what's up",
            "good afternoon",
            "morning",
            "good evening",
        }
    )
)
```

The signal -- a set of strings to exclude -- is now expressed directly.

### Diagnostic Questions

| Question | If Yes |
|----------|--------|
| Am I repeating the same function wrapper N times? | Use a collection-based alternative |
| Am I writing a loop body that is nearly identical each iteration? | Extract the varying part |
| Do I have multiple variables that follow the same pattern? | Use a data structure |

## Flat Is Better Than Nested

Humans are poor at counting brackets. Nested expressions force readers
to build a mental stack. Flat pipelines read top-to-bottom.

Avoid:
```python
result = tuple(map(increment, [1, 2, 3]))
```

The reader must parse inside-out: list, then map, then tuple.

Prefer:
```python
result = gamla.pipe(
    [1, 2, 3],
    gamla.map(increment),
    tuple,
)
```

Data flows top-to-bottom. Each step is a single line.

### When Nesting Is Acceptable

Short, simple expressions where the nesting depth is 1-2 levels and
the meaning is immediately obvious. Example: `len(set(items))`.

The threshold: if a reader needs to pause to parse the structure,
flatten it.

## Work on the Single Case

Minimize transitions between the collection space and the single-value
space. When multiple transformations apply to each element, compose
them before mapping.

Avoid:
```python
result = compose_left(map(f1), map(f2), map(f3))
```

Three separate traversals of the collection, each requiring the reader
to track both the collection level and the element level.

Prefer:
```python
result = map(compose_left(f1, f2, f3))
```

One traversal. The composed function operates on a single element,
which is easier to reason about. This also simplifies the code and
removes cognitive overhead -- it is easier to hold one object in mind
than a plurality.

## Point-Free (Tacit) Programming

Point-free style avoids naming noise and signature duplication. When a
name adds no value or readability, working at the function level instead
of the value level removes that noise.

### Recognizing Pointless Names

Avoid:
```python
(x for x in people if my_filter(x))
```

`x` is a meaningless name. Even `person` would repeat information
already present in `people` (three times: twice for the variable name,
once implied by the collection name).

Prefer:
```python
filter(my_filter, people)
```

The idea is expressed cleanly with considerably less noise.

### When to Use Point-Free

| Situation | Recommendation |
|-----------|---------------|
| Lambda wraps a single function call | Go point-free |
| Variable name is meaningless (`x`, `item`, `elem`) | Go point-free |
| Variable name restates the collection name | Go point-free |
| Variable name carries meaningful domain info | Keep the name |
| Logic is complex and benefits from intermediate names | Keep the name |

## Eta Reduction

Eta reduction removes variables that appear at the ends of an
implementation, lifting from the value level to the function level.

### Pipeline Input Reduction

When a pipeline needs an input from the top, return the composition
instead of wrapping in a function.

Avoid:
```python
def my_pipeline(x):
    return gamla.pipe(x, f, g)
```

`x` appears only at the start and is threaded through. The function
is just a named composition.

Prefer:
```python
my_pipeline = gamla.compose_left(f, g)
```

### Signature Simplification

When some arguments are used at the top and others at the bottom of a
pipeline, return a function to simplify the signature. This eliminates
redundant currying and redundant arguments.

Avoid:
```python
@gamla.curry
def my_func(x, y, z):
    return gamla.pipe(z, run(x), run_also(y))

gamla.pipe(
    ...,
    my_func(x, y),
)
```

Three problems: redundant `@curry`, redundant argument `z` (it just
flows through the pipe), and an unnecessary pipe stage.

Prefer:
```python
def my_func(x, y):
    return gamla.compose_left(run(x), run_also(y))

gamla.pipe(
    ...,
    my_func(x, y),
)
```

The resulting function applies directly to the pipe input. No currying
needed, one fewer argument, one fewer stage.

## Pipeline Construction vs. Running

Do not mix constructing a pipeline with running it. When a constant
value determines how the pipeline should behave, resolve that decision
before the pipeline runs -- similar to hoisting a constant out of a loop.

### The Symptom

`when`/`unless`/`ternary` combined with `always`/`just` means a
pipeline is being constructed mid-execution.

Avoid:
```python
my_function = gamla.ternary(gamla.just(my_var), f, g)
```

This builds conditional logic at runtime around a value that is already
known.

Prefer:
```python
my_function = f if my_var else g
```

The decision is made once, before any data flows.

## Purity and Immutability

Pure functions (no side effects) and immutable data make debugging,
caching, and serialization straightforward.

### Principles

| Principle | Rationale |
|-----------|-----------|
| Avoid global state | Invisible coupling between distant code |
| No mutable classes | Often just hidden global variables |
| Minimize in-function state | Name logic as static functions instead of variables |
| Use pure functions | Predictable, testable, composable |

### Store Facts, Compute Derived Values Lazily

Store raw facts. Compute derived attributes on demand rather than
caching them eagerly. This eliminates stale-data bugs and reduces
mutable state.

### No Global Caching

Global caches (`functools.lru_cache` at module level) are mutable global
state in disguise. They make performance "sometimes" better with unclear
rules, complicate optimization and latency testing, require reloading
code for a "clean slate," and are never garbage collected.

Avoid:
```python
@functools.lru_cache
def something_slow(x):
    ...

@gamla.curry
def something_fast(x, element):
    return something_slow(x) + element

def loop(x, elements):
    return tuple(map(something_fast(x), elements))
```

Prefer: factor the slow computation outside the hot path.

```python
def something_slow(x):
    ...

@gamla.curry
def something_fast(slow_result, element):
    return slow_result + element

def loop(x, elements):
    slow_result = something_slow(x)
    return tuple(map(something_fast(slow_result), elements))
```

The slow result is computed once and passed explicitly. No hidden
global mutable state.

## Currying Conventions

Non-unary functions used in compositions benefit from correct argument
ordering. The more fundamental an argument is to the function's
operation, the earlier it should appear in the signature.

### Argument Order Rule

Higher-order functions place the function parameter first (like `map`
and `filter`). The "data" argument -- the one flowing through a
pipeline -- comes last.

Avoid:
```python
@gamla.curry
def foo(x, y):
    ...

gamla.compose_left(foo(y=2), bar)
```

Keyword arguments in composition are noisy and fragile.

Prefer:
```python
@gamla.curry
def foo(y, x):
    ...

gamla.compose_left(foo(2), bar)
```

Positional partial application reads cleanly in pipelines.

## Avoiding Ambiguity

Optionality, nullability, and default arguments introduce multiple
ways to call a function. This makes reasoning about behavior and
refactoring harder.

### Default Arguments

Default arguments interact poorly with currying:

```python
@gamla.curry
def do_something(x, y=None):
    if y is not None:
        ...
```

Is `do_something(x)` a call with `y=None`, or a partial application
waiting for `y`? Defaults make this ambiguous.

Defaults also enable bloated signatures:

```python
def pizza_spec(size, topping, part="entire-pizza", crust=None, beverage=None):
    ...
```

Each optional parameter multiplies the number of possible call patterns.
Refactoring requires considering all of them.

### Alternative: Separate Functions

Instead of one function with defaults, create focused functions:

Avoid:
```python
def calculate(x, addition, multiplication=1):
    return (x + addition) * multiplication
```

Better (currying):
```python
@gamla.curry
def calculate(multiplication, addition, x):
    ...

addition = calculate(1)  # Users not needing multiplication use this
```

Best (decomposition):
```python
def multiplication(x, y):
    return x * y

def addition(x, y):
    return x + y
```

Smaller signatures, clearer intent, no ambiguity about which parameters
are required.

### Decomposition Example

A large function with optional parameters can usually be split into
smaller composable functions:

```python
def topping_request(topping, part):
    ...

combine_requirements = gamla.compose_left(gamla.mapcat(dict.items), dict)

order = combine_requirements(
    topping_spec("mushrooms", "all"),
    size_spec("medium"),
)
```

If no crust information is needed, it simply is not there. It is also
now obvious that `part` refers to the topping, not the entire order.

## Redundant Pipes

A pipe with a single function is noise.

Avoid:
```python
gamla.pipe(x, f)
```

Prefer:
```python
f(x)
```

Exception: when the function is a complex expression with brackets,
a pipe can improve readability by separating data from transformation.

Avoid:
```python
expression_with_brackets(and_nesting)(x)
```

Prefer:
```python
gamla.pipe(
    x,
    expression_with_brackets(and_nesting),
)
```

The pipe separates the input from the operation, making both easier to read.

## Pattern Summary

| Pattern | Benefit |
|---------|---------|
| Flatten nested expressions | Top-to-bottom readability |
| Compose before mapping | Single traversal, simpler logic |
| Point-free style | Less noise, cleaner composition |
| Eta reduction | Simpler signatures |
| Separate construction from running | Clear decision points |
| Factor out slow computations | Explicit data flow, no global cache |
| Data argument last | Clean partial application |
| Separate functions over defaults | Smaller signatures, no ambiguity |
