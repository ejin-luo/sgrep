## Config format

This document provides examples for the .yaml format for specifying rules to sgrep. It does not cover the patterns themselves; for information on the query sytnax see [patterns.md](patterns.md)

If you want run these rules or ship them to your team, rather than write them, check out the easy-to-use [bento.dev](https://bento.dev)

## Simple Example

```yaml
rules:
  - id: eqeq-is-bad
    pattern: $X == $X
    message: "$X == $X is always true!"
    languages: [python]
    severity: ERROR
```

This is a minimal rule. It will be run on any Python code, and will output the rule `id` "eqeq-is-bad" along with the `message` and the line and column range the match occurred on. So it will match code like `1 == 1` and output the message: "Dude, 1 == 1 is always true!"

## Complex Example
```
rules:
  - id: eqeq-is-bad
    patterns:
      - pattern-not-inside: |
          def __eq__(...):
              ...
      - pattern-not-inside: assert(...)
      - pattern-not-inside: assertTrue(...)
      - pattern-not-inside: assertFalse(...)
      - pattern-either:
          - pattern: $X == $X
          - pattern: $X != $X
          - patterns:
            - pattern-inside: |
                 def __init__(...):
                      ...
            - pattern: self.$X == self.$X
      - pattern-not: 1 == 1
    message: "useless comparison operation `$X == $X` or `$X != $X`; if testing for floating point NaN, use `math.isnan`, or `cmath.isnan` if the number is complex."
    languages: [python]
    severity: ERROR
```

The main difference here is that we are composing patterns together. Every pattern is implicitly ANDed together, i.e.:

```
    ...
    patterns:
      - pattern: foo()
      - pattern: foo(1)
```

Will match nothing, because foo() and foo(1) can never occur together. If we made the first pattern `foo(...)`, the rule will work, although the pattern is unecessary.

## Pattern Composition Operators

### basic operators

There are several operators that can be used in a rule. At the top level, there are three:

- `pattern`: The rule will only fire if this pattern is found.
- `pattern-either`: (logical OR) Nest multiple other patterns under this; any of those patterns will count as a match.
- `pattterns`: (logical AND) You can put multiple patterns under this to create nested, implicitly-ANDed instructions.

### filter operators

Filters: there are several operators that act as filters to remove results you don't want. They must be combined with `pattern` operator in order to see results; e.g., if you have a rule with just `patttern-not`, nothing will fire.

- `pattern-not`: This rule will filter out any cases where the pattern is found.
- `pattern-inside`: Filter out results that do not lie inside this specified pattern. Useful for specifying a function that this behavior must occur in, for instance.
- `pattern-not-inside`: Opposite of `pattern-inside`; this will filter out any following pattern results that are *not* inside the specified pattern.
- `pattern-where-python`: Uses a python expression to decide whether or not to filter out this result. Variables are passed to your python expression Python in an array called `vars`. **Running rules from other people that use this filter is dangerous as it can allow arbitrary code exeuction. If you have a rule using Python, you must pass the flag  `--dangerously-allow-arbitrary-code-execution-from-rules`.** Example:
  ```sgrep
    patterns:
      - pattern: $F = django.db.models.FloatField(...)
      - pattern-where-python: "'price' in vars['$F']"
  ```

## Schema

Each rule object has these fields:

| Field     | Type          | Description                                                                                                        | Required |
| --------- | ------------- | ------------------------------------------------------------------------------------------------------------------ | -------- |
| id        | string        | None unique check-id that should be descriptive and understandable in isolation by the user. e.g. `no-unused-var`. | Y        |
| `pattern` or `patterns` or `pattern-either`   | string        | See example patterns in this document.                                                                                        | Y        |
| message   | string        | Description of the rule that will be output when a match is found.                                                 | Y        |
| languages | array<string> | Languages the check is relevant for. Can be python or javascript.                                                  | Y        |
| severity  | string        | Case sensitive string equal to WARNING, ERROR                                                                  | Y        |
