# MiniTemplate Public Packet

## Overview

Build `minitemplate.py`, a text template engine supporting variable substitution, conditionals, loops, block definitions, and variable assignment. It should handle common template patterns: injecting values with `{{ var }}`, branching with `{% if %}`, iterating with `{% for %}`, and organizing reusable sections with `{% block %}`.

This task is designed around the distinction between rendering a single template correctly and maintaining consistency across multiple renders, error recovery, and template compilation. Individual template features should work on their own, but the engine is only complete if parsed templates can be re-rendered with different contexts, undefined variables raise proper errors without corrupting the template, and conditionals/loops/blocks compose correctly.

The implementation language is Python 3.11. Place `minitemplate.py` at the root of your solution directory. Use only the Python standard library.

Public API:

```python
from minitemplate import Template, Environment
from minitemplate import TemplateSyntaxError

# Simple template
t = Template('Hello {{ name }}!')
print(t.render(name='Alice'))          # → Hello Alice!

# Conditionals and loops
t = Template('{% if show %}{% for i in items %}{{ i }}{% endfor %}{% endif %}')
print(t.render(show=True, items=[1,2,3]))  # → 123

# Environment (for advanced use)
env = Environment()
t = env.from_string('{{ greeting }} {{ name }}')
print(t.render(greeting='Hi', name='Bob'))  # → Hi Bob
```

The benchmark does not inspect private implementation details.

## Feature Set

The product has six feature modules:

1. Variable substitution — `{{ var }}`, `{{ obj.attr }}`, and `{{ d.key }}` with dot-notation access.
2. Conditional blocks — `{% if cond %}`, `{% elif cond %}`, `{% else %}`, `{% endif %}` with comparison operators.
3. Loop blocks — `{% for item in seq %}`, `{% else %}`, `{% endfor %}`.
4. Block definitions — `{% block name %}`, `{% endblock %}` for reusable content sections.
5. Variable assignment — `{% with name = value %}`, `{% endwith %}` scoped variable binding.
6. Environment and error handling — `Environment` class for compilation, plus `TemplateSyntaxError` and `UndefinedError`.

These modules are intentionally interdependent. Variable substitution feeds values into conditionals and loops. Block definitions interact with template rendering. The `Environment` class provides an alternative compilation path that produces the same `Template` objects. Errors during parsing (syntax) and rendering (undefined variables) must be properly isolated.

## Global Invariants

- `Template.render()` must produce the same output for the same context regardless of how many times it is called.
- A `Template` instance can be re-rendered with different contexts and each result is correct and independent.
- `Environment.from_string()` produces `Template` instances with the same `render()` behavior as directly-constructed ones.
- Undefined variables in `{{ }}` are silently rendered as empty strings.
- Undefined variables in `{% if %}` conditions are treated as falsy.
- Syntax errors (`{% if %}` without `{% endif %}`, etc.) must raise `TemplateSyntaxError` at parse time.
- Block definitions in templates define named content sections that render their body content when the template is rendered.
- `{% with %}` scoped variables shadow any outer variables of the same name only within the block.
- Dot notation `obj.attr` first tries attribute access, then key/index access.

## Template Syntax

### Variables

```
{{ name }}
{{ user.name }}
{{ d.key }}
```

Render-time keyword arguments become template variables. Dot notation accesses nested attributes or dict keys.

### Conditionals

```
{% if user %}
  Hello {{ user }}!
{% elif guest %}
  Welcome guest.
{% else %}
  Please log in.
{% endif %}
```

Supported comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`. Truthiness follows Python rules. Undefined variables in conditions raise `UndefinedError`.

### Loops

```
{% for item in items %}
  {{ item }}
{% else %}
  No items.
{% endfor %}
```

The `{% else %}` branch renders if the sequence is empty. The loop variable shadows any outer variable of the same name only within the loop body.

### Block Definitions

```
{% block header %}
  Default header content.
{% endblock %}
```

Blocks define named sections. The block's body content is rendered where the block appears.

### Variable Assignment

```
{% with x = 5 %}
  Value: {{ x }}
{% endwith %}
```

Creates a scoped variable binding. The variable is only accessible within the `{% with %}...{% endwith %}` block.

## API

### `class Template`

#### `Template(source)`

Parse a template string. Raises `TemplateSyntaxError` on syntax errors.

#### `Template.render(**kwargs)`

Render the template with keyword arguments as variables. Returns the rendered string. Undefined variables are rendered as empty strings.

### `class Environment`

#### `Environment()`

Create a template environment.

#### `Environment.from_string(source)`

Compile a template string. Returns a `Template` instance. Equivalent behavior to `Template(source)`.

### Exceptions

- `TemplateSyntaxError` — raised at parse time for malformed templates. Inherits from `Exception`.

## Error Behavior

- Malformed templates (`{% if %}` without `{% endif %}`, `{% for %}` without `{% endfor %}`, unclosed blocks) must raise an error at parse time.
- Undefined variables in `{{ }}` are silently rendered as empty strings (no error).
- After a parse error, subsequent valid template creation must succeed.
- The exact error message text is not part of the public API.

## Non-Goals

- No template inheritance (`{% extends %}`).
- No filters (`| upper`, `| lower`).
- No comments (`{# ... #}`).
- No includes (`{% include %}`).
- No arithmetic expressions in templates (`{{ a + b }}`).
- No function calls in templates (`{{ func() }}`).
- No HTML escaping or auto-escaping.
- No custom delimiters.

## Evaluation Style

Hidden tests are split into two scores:

- Unit tests exercise one feature module at a time using short Python snippets.
- System tests exercise interactions across at least two modules.

System tests are labeled by dimension: `cross_feature_dataflow`, `state_accumulation`, `global_invariant`, `error_atomicity`, `operation_order_sensitivity`, `boundary_crossing`.
