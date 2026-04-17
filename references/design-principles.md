# Design Principles

Sources: Production functional codebases, Clean Code principles, FP community
conventions

Covers: code conciseness, commenting standards, naming conventions, context-free
design, single responsibility, dead code elimination, code review process.

## Conciseness

Bug occurrence correlates with code volume. Long implementations signal
duplication or design problems. Scope size correlates with the number of
possible interactions between elements -- meaning cognitive load grows with
scope length.

Code length is perhaps the only objective metric for evaluating design.

### Implications

- Delete code often. Unused code is a liability, not an asset.
- Duplication is the opposite of reuse. Eliminating it enables centralized
  optimization -- one improvement benefits many call sites.
- When an implementation feels long, look for hidden duplication or
  opportunities to decompose.

### Measuring Conciseness

Do not count conciseness by character count alone. Concise code has fewer
concepts, fewer moving parts, fewer places where things can go wrong. A 3-line
function with clear intent beats a 1-line expression that requires mental
unpacking.

## Code Comments

Comments are read more than once. They deserve the same care as production code
-- no typos, correct capitalization, proper punctuation.

### When to Comment

| Situation                                 | Action                        |
| ----------------------------------------- | ----------------------------- |
| Implementation is self-explanatory        | No comment needed             |
| Reader might misunderstand the why        | Comment the reasoning         |
| Complex algorithm with non-obvious steps  | Brief explanation of approach |
| Workaround for external bug or limitation | Cite the issue/constraint     |

### When Not to Comment

| Situation                           | Action                                  |
| ----------------------------------- | --------------------------------------- |
| Comment restates what the code does | Delete it                               |
| Code is unclear                     | Refactor the code instead of commenting |
| Comment explains a name             | Choose a better name                    |
| Commented-out code                  | Delete it -- version control exists     |

Most code should be self-explanatory. When it is not, prefer refactoring over
commenting. Comments are a last resort to illuminate something a reader might
not immediately grasp.

## Consistency and Autoformatting

Inconsistencies distract readers and waste reviewer time on style debates. The
solution: autoformat everything, including configurations and markdown.

Prefer opinionated autoformatters that rewrite code from the abstract syntax
tree. Automatic consistency removes redundant degrees of freedom -- nobody
argues about bracket placement when the formatter decides.

### Recommended Formatters

| Language   | Formatter  | Config                   |
| ---------- | ---------- | ------------------------ |
| Python     | `black`    | No config (use defaults) |
| JavaScript | `prettier` | No config (use defaults) |

The key: zero configuration. Configurable formatters reintroduce the debates
they are supposed to eliminate.

## Naming Conventions

### When to Name

Factor out and name a function when:

1. The function body clutters the surrounding context, making the higher-level
   logic harder to follow.
2. The same logic appears in multiple places -- naming enables reuse.

### When Not to Name

1. The name is approximately the same length as the implementation. Naming adds
   indirection without adding clarity.

   Avoid:
   ```python
   filter_larger_than_1 = gamla.filter(operator.lt(1))
   ```
   The name is no more informative than the code itself.

2. A single-export file where the name would repeat the filename.

   Avoid:
   ```javascript
   // In f.js
   const f = () => true;
   export default f;
   ```

   Prefer:
   ```javascript
   // In f.js
   export default () => true;
   ```

### Avoid "Utils"

"Utils" is a lazy naming convention that becomes meaningless quickly. It creates
non-trivial dependency tangles across projects. If something is truly a utility,
it belongs in a shared library (open source or internal), not in a `utils.py` or
`utils.js` file in an application.

## Context-Free Code

A function's name and description should refer only to the logic it contains,
never to the context where it will be used. Context-free functions are reusable
and independently understandable.

### Functions

Avoid:

```python
def make_price_slightly_higher(price: int) -> int:
    return price + 1
```

The name leaks the caller's domain. A reader does not need that context to
understand what the function does, and the name prevents reuse.

Prefer:

```python
def increment(n: int) -> int:
    return n + 1
```

### Filenames

The same principle applies to filenames. If a directory scopes a domain, do not
repeat the domain in filenames inside it.

Avoid:

```
fruit/fruit_apple.py
fruit/fruit_tomato.py
```

Prefer:

```
fruit/apple.py
fruit/tomato.py
```

## Single Responsibility

Large blocks of code deserve scrutiny. Ask:

- Is more than one thing happening here?
- Can a subset of this logic receive a clear name?
- Is the order of operations arbitrary (suggesting independent concerns)?

If any answer is yes, factor out and compose rather than nest.

### Self-Disable Pattern

A function that starts with a guard clause checking whether its core logic
should run is doing two things: deciding and executing. Separate them.

Avoid:

```python
def f(a, b, c):
    if check(a):
        return
    ...  # core logic

f(a, b, c)
```

`f` is coupled to `check`. They cannot vary independently.

Prefer:

```python
def f(a, b, c):
    ...  # core logic

if not check(a):
    f(a, b, c)
```

Now `f` and `check` are oblivious to each other. Each is simpler and
independently testable.

## No Dead Code

Remove all code not connected to the running application:

| Pattern                      | Action                                      |
| ---------------------------- | ------------------------------------------- |
| "Preparing for the future"   | Delete -- predictions are unreliable        |
| "This might be useful again" | Delete -- version control preserves history |
| Commented-out code blocks    | Delete immediately                          |
| Debugging helpers            | Move to an external library                 |
| Unused imports or variables  | Delete                                      |

Dead code creates false signals during maintenance. Readers assume code exists
for a reason and waste time understanding paths that never execute.

## Code Review Process

### Flow

1. Author opens a merge request and assigns a reviewer.
2. Reviewer comments and assigns back to author.
3. Author resolves all comments (by fixing or replying).
4. Iterations continue until the reviewer approves (or writes "LGTM").
5. Author merges (squashing commits unless every individual commit is clean and
   has a valid description).

### Reviewer Responsibilities

The reviewer is responsible for two goals that rarely conflict:

1. **Maintain code health** -- code health should never decrease, not even
   locally. Regressions require a fix before merge (or an explicit `TODO` if
   time-critical).
2. **Help the author merge quickly** -- unmerged code is technical debt. Find
   the minimal stopping point that maintains code health while preserving
   correctness.

### Reviewer Anti-Patterns

| Anti-Pattern                                        | Why It Hurts                       |
| --------------------------------------------------- | ---------------------------------- |
| "You fixed this, now fix that too" (unrelated code) | Expands scope beyond the PR        |
| "Use this new pattern everywhere"                   | Turns a focused PR into a refactor |
| "Since you wrote one test, cover all the others"    | Unrelated to the change            |

These observations are fine as optional suggestions ("consider...") but should
not block merge.

### Author Responsibilities

- Squash commits on merge for clean history.
- Resolve every comment -- either fix or explain why not.
- Keep PRs focused on a single concern.

## Interaction Between Principles

These principles reinforce each other:

| Principle                | Enables                                       |
| ------------------------ | --------------------------------------------- |
| Conciseness              | Reduces naming decisions, simplifies reviews  |
| Context-free naming      | Enables reuse, reduces coupling               |
| Single responsibility    | Enables conciseness, clearer names            |
| No dead code             | Keeps codebase honest, reduces cognitive load |
| Autoformatting           | Eliminates style debates in reviews           |
| Meaningful comments only | Keeps signal-to-noise ratio high              |

When principles appear to conflict, conciseness usually wins -- shorter code
with clear intent is almost always better than longer code with extensive
documentation.
