---
name: clean-coding
description: >
  Apply clean code practices during design, implementation, and review: DRY, SOLID,
  prefer functional style over OOP when suitable, strict typing, and Python tooling
  (ruff, pyright, black). Use when writing or refactoring code, reviewing PRs/diffs,
  naming APIs, reducing complexity, or when the user mentions clean code, Uncle Bob,
  SOLID, DRY, YAGNI, KISS, pure functions, or lint/typecheck hygiene.
---

# Clean Coding

Operate as a disciplined craftsman (Uncle Bob, Beck, Fowler, Hunt & Thomas, Okasaki-style FP).
Every change should leave the codebase **clearer, smaller in concept count, and safer under types**.

## When this skill applies

- Writing new modules, functions, or APIs
- Refactoring, reviews, “make this cleaner”
- Choosing FP vs OOP, layering, error handling
- Setting or respecting ruff / pyright / black

## Non-negotiable principles

### 1. Names reveal intent

- Names are documentation. Prefer `elapsed_seconds` over `t`, `is_eligible` over `flag`.
- Functions are verbs/phrases: `load_user`, `compute_invoice_total`.
- Booleans read as predicates: `has_permission`, `can_retry`.
- Avoid encodings (`IUser`, `strName`) and vague buckets (`data`, `info`, `manager`, `util` dumping grounds).

### 2. Functions stay small and do one thing

- One level of abstraction per function (step-down rule).
- Prefer ≤ ~20 lines of real logic; extract when nesting or comments explain “blocks”.
- Command–query separation: mutators don’t return domain data; queries don’t mutate.
- Prefer pure functions: same inputs → same outputs; push I/O to the edges.

### 3. DRY — but not premature abstraction

- **Rule of three**: duplicate once if unclear; third time, abstract with a name that earns its keep.
- DRY is about **knowledge**, not characters. Two identical lines with different reasons may stay separate.
- Don’t invent shared helpers that couple unrelated domains (“utils.py god module”).
- Prefer duplicating a thin adapter over a wrong shared base class.

### 4. KISS & YAGNI

- Simplest design that passes tests and expresses intent.
- No speculative generality, config flags, or plugin systems “for later”.
- Delete dead code; don’t comment it out.

### 5. SOLID (use judgment — not ceremony)

| Principle | Practical meaning |
|-----------|-------------------|
| **S**ingle Responsibility | One reason to change per module/type. Split “loads + validates + emails”. |
| **O**pen/Closed | Extend via new functions/types/composition, not endless `if kind ==`. |
| **L**iskov | Subtypes must honor contracts; no surprising overrides. Prefer composition. |
| **I**nterface Segregation | Small, role-specific protocols/interfaces; callers shouldn’t depend on fat APIs. |
| **D**ependency Inversion | High-level policy depends on abstractions (protocols/callables), not concrete I/O. |

Inject dependencies (repos, clocks, RNG, HTTP) as parameters or narrow protocols — enables tests without mocks-of-everything.

### 6. Prefer functional style; OOP only when it pays

**Default to FP-friendly code:**

- Pure functions, immutable data (`frozen` dataclasses / `NamedTuple` / typed dicts used as values).
- Explicit data transformation pipelines: map/filter/reduce, comprehensions when readable.
- Errors as values when it clarifies control flow (`Result`-like, or clear raise at boundaries).
- Closures and small callables over strategy class hierarchies.
- Composition of functions over deep inheritance.

**Use OOP / classes when necessary:**

- True mutable identity with invariants (ORM entities, connection pools, complex UI state).
- Defining a **stable boundary** (Protocol + few implementations) for DI.
- Resource lifetimes that need structured cleanup (`contextmanager` still preferred when enough).
- When the domain language is naturally object-ish *and* behavior clusters with data.

**Avoid:** anemic “Manager/Helper/Service” class forests, inheritance for code reuse, god objects, mutable global singletons.

Python idioms that fit this stance:

```python
from dataclasses import dataclass
from collections.abc import Callable, Iterable, Sequence
from typing import TypeVar

T = TypeVar("T")
U = TypeVar("U")

@dataclass(frozen=True, slots=True)
class Money:
    cents: int
    currency: str

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(self.cents + other.cents, self.currency)

def pipe(value: T, *fns: Callable[[T], T]) -> T:
    for fn in fns:
        value = fn(value)
    return value
```

### 7. Types are part of the design

- Annotate public functions, module APIs, and non-obvious locals.
- Prefer precise types: `Sequence[UserId]`, `Literal["asc", "desc"]`, `NewType`, enums — not bare `dict`/`Any`.
- Use `Protocol` for structural dependencies instead of ABC trees when one method suffices.
- Use `T | None` only when `None` is meaningful; don’t hide errors in `None`. Prefer `T | None` over `Optional[T]` (PEP 604).
- Keep pyright/mypy clean: no unexplained `# type: ignore`; if needed, narrowest code + reason.

### 8. Errors, boundaries, and side effects

- Validate at system boundaries (HTTP, CLI, file, queue). Inside, trust typed invariants.
- Fail fast with specific exceptions or result types; don’t swallow.
- Log at edges with context; pure core stays quiet.
- Time, randomness, network, filesystem = parameters or ports (testability).

### 9. Comments & structure

- Comment **why**, never what the next line obviously does.
- Delete comments that restate code; rewrite code until the comment is unnecessary.
- TODOs need owner/ticket or they are debt — prefer fixing or filing.
- Modules: cohesive packages; public API via intentional `__all__` or package exports.
- Tests document behavior; messy tests signal messy design.

### 10. Formatting & lint are automatic, not debates

For Python projects, treat these as the default quality bar:

| Tool | Role |
|------|------|
| **black** | Uncompromising format (or ruff-format) — no style bikeshedding |
| **ruff** | Lint + import sort + many bugbear/pyflakes/isort rules — fast feedback |
| **pyright** (or mypy strict) | Static types — catch whole classes of bugs before runtime |

When editing Python:

1. Match existing tool config (`pyproject.toml`, `ruff.toml`, `pyrightconfig.json`).
2. Don’t disable rules broadly to silence a smell — fix the code or scope `# noqa` with reason.
3. After substantive edits, run the project’s check commands if available, e.g.:

```bash
ruff check --fix .
ruff format .
pyright
# or: black . && ruff check . && pyright
```

4. Prefer ruff rules that encode clean code: complexity (`C901`), simplify (`SIM`), bugbear (`B`), pyupgrade (`UP`), isort (`I`), pylint-lite (`PL`).
5. Keep functions under complexity limits; if ruff complains, extract — don’t raise the limit casually.

Suggested minimal `pyproject` intent (adapt to repo; don’t fight established config):

```toml
[tool.black]
line-length = 88

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "C4", "C901", "PL", "RUF"]

[tool.pyright]
typeCheckingMode = "standard"  # or "strict" when the codebase allows
```

## Workflow while coding

1. **Understand the change** — one clear behavior or fix.
2. **Sketch types and data flow** before classes or frameworks.
3. **Implement the pure core first**, then adapters (I/O).
4. **Name ruthlessly**; rename when understanding improves.
5. **Remove duplication of knowledge** after the third repetition or when a name is obvious.
6. **Run format + lint + types** before considering the work done.
7. **Read the diff as a reviewer**: Would Uncle Bob merge this? Is there a smaller design?

## Review checklist (use on every non-trivial change)

- [ ] Names express intent; no cryptic abbreviations
- [ ] Functions/modules have one reason to change
- [ ] No speculative abstraction; no god utils
- [ ] Side effects at edges; core mostly pure
- [ ] FP preferred; classes justified
- [ ] Types precise; pyright-clean
- [ ] Errors explicit; no bare `except:`
- [ ] Tests or clear manual verification for behavior
- [ ] ruff/black clean; no unjustified ignores
- [ ] Diff is smaller/clearer than before (boy scout rule)

## Red flags — stop and refactor

- Long parameter lists → parameter object or split function
- Deep nesting → guard clauses, extract, polymorphism/match
- Feature envy / shotgun surgery → wrong boundaries
- Boolean flag parameters that switch behavior → split functions
- Inheritance deeper than 1–2 levels → composition
- `Any`, untyped dicts crossing layers → model the data
- Copy-paste with one field different → parameterization only if domain-same
- Comments explaining bad code → rewrite the code

## Language-agnostic craft (still apply outside Python)

- Boy Scout Rule: leave the campground cleaner.
- Law of Demeter: talk to immediate friends, not `a.b().c().d()`.
- Tell, don’t ask (when objects own behavior); for pure data, transforms are fine.
- Optimize for **reader time**, not writer cleverness.
- Consistency with local codebase beats personal taste — then improve local standards deliberately.

## How to respond when this skill is active

- Prefer concrete refactors and patches over lectures.
- When proposing design, state **FP vs OOP choice and why**.
- Call out SOLID/DRY violations with a minimal better shape.
- Wire or respect ruff/pyright/black rather than hand-formatting.
- Do not expand scope into large rewrites unless asked; boy-scout within the change.

## Anti-patterns this skill rejects

- “Enterprise” layers with no behavior
- Premature microservices / patterns from blog posts without need
- Mock-heavy tests that freeze implementation
- Turning off type checkers to ship
- Clever one-liners that obscure data flow
- OOP theater (singleton factories of abstract builder visitors) for simple scripts
---

## Quick reference card

```
DRY     → one authoritative representation of each piece of knowledge
KISS    → simplest design that works and reads well
YAGNI   → don’t build what you don’t need now
SOLID   → boundaries & dependencies that age well
FP>OOP  → pure functions + immutable data by default; classes for identity/invariants/ports
Types   → design aloud; pyright is the critic
Tools   → black/ruff format+lint, pyright types — always green
Scout   → every PR slightly cleaner than main
```
