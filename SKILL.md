---
name: code-style-guide
description: |
  Functional programming code style guide emphasizing conciseness, purity,
  composition, and point-free patterns. Covers Python and JavaScript.
  Synthesized from production functional codebases.

  Trigger phrases: "code style", "code review", "functional programming",
  "point-free", "tacit programming", "eta reduction", "currying",
  "function composition", "pipe", "compose", "immutability", "purity",
  "clean code", "refactor", "code smell", "naming conventions",
  "default arguments", "Python style", "JavaScript style", "gamla"
---

# Functional Code Style

## Core Philosophy

1. **Conciseness** -- Bug occurrence correlates with code volume. Scope size
   correlates with cognitive load. Code length is the most objective design
   metric. Write less code, delete code often.
2. **Purity and immutability** -- Pure functions make debugging, caching, and
   serialization straightforward. Avoid mutable state, global caches, and side
   effects.
3. **Composition over nesting** -- Build pipelines of small functions rather
   than deeply nested expressions. Flat structures are easier to read.
4. **Signal over noise** -- Every token should carry meaning. Remove redundant
   names, arguments, and abstractions that restate what the code already says.
5. **Context-free functions** -- Name functions by what they do, not where they
   are used. Context-free code is reusable and readable.

## Quick Decisions

### Should I name this function?

| Signal                                       | Decision                     |
| -------------------------------------------- | ---------------------------- |
| Implementation clutters surrounding context  | Name it                      |
| Avoids repeating yourself                    | Name it                      |
| Name is similar length to the implementation | Inline it                    |
| Single-export file, name repeats filename    | Use anonymous default export |

### Should I add a comment?

| Signal                               | Decision                   |
| ------------------------------------ | -------------------------- |
| Code is self-explanatory             | No comment                 |
| Reader might not understand why      | Add comment explaining why |
| Comment describes what the code does | Refactor the code instead  |

### Should I use a default argument?

Almost never. Defaults create ambiguity, complicate currying, and hide call-site
variation. Use separate functions or curried partial application. See
`references/functional-patterns.md`.

### How should I order function arguments?

More fundamental arguments first. Higher-order functions place the function
parameter first. The "data" argument (the one flowing through a pipeline) comes
last.

## Code Reviews

Reviewers protect code health without scope-creeping:

- Code health should never decrease, not even locally
- Unmerged code is technical debt -- help authors merge quickly
- Avoid expanding project scope beyond what the author intended
- Flag regressions; optional suggestions are fine when marked as such
- Authors squash commits on merge unless every individual commit is clean

## Language Quick Reference

### Python

| Rule                                           | Rationale                                           |
| ---------------------------------------------- | --------------------------------------------------- |
| Use `black` with no config                     | Opinionated formatting removes debates              |
| `_prefix` for module-private names             | Convention signals intent                           |
| Prefer `frozenset`, `tuple` over `set`, `list` | Immutability by default                             |
| Import modules, not inner names                | Except `typing` -- avoids hidden coupling           |
| Use `asyncio` over threads/processes           | GIL makes threads ineffective for CPU work          |
| Avoid `Optional`, `Union` in types             | They increase ambiguity                             |
| Use `Tuple[X, ...]` for homogeneous tuples     | Different from `Tuple[X]` (single element)          |
| Avoid `defaultdict`                            | Encourages mutation patterns                        |
| Retain stable ordering in outputs              | Prevents flaky tests; use `frozenset` for unordered |

### JavaScript

| Rule                                         | Rationale                                       |
| -------------------------------------------- | ----------------------------------------------- |
| Use `prettier` with no config                | Opinionated formatting removes debates          |
| `const` always, `let` if needed, never `var` | Immutability by default                         |
| Functional components only in React          | No `Component` class                            |
| Arrow functions, body-less when possible     | `x => x+1` over `x => { return x+1; }`          |
| Prefer `export default`                      | Avoids name duplication between file and export |
| Keyworded args for non-unary functions       | `({a, b}) => ...` prevents positional errors    |

## Reference Files

| File                                | Contents                                                                                                        |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `references/design-principles.md`   | Conciseness, comments, naming, context-free code, single responsibility, no dead code, code reviews             |
| `references/functional-patterns.md` | Point-free style, eta reduction, currying, pipelines, purity, immutability, signal-to-noise, avoiding ambiguity |
