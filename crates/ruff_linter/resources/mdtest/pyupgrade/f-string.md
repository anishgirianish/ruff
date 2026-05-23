# `f-string` (`UP032`)

Soundness fixes for [#15874](https://github.com/astral-sh/ruff/issues/15874): a `str.format` call is
only converted when the f-string behaves like the original. Conversions that can change behavior
become unsafe fixes; conversions that Python rejects, or that an f-string reads differently, are
skipped.

```toml
lint.select = ["UP032"]
```

## Dropped arguments emit an unsafe fix

An f-string only evaluates the arguments it interpolates, so dropping one that can change behavior is
unsafe.

<!-- snapshot-diagnostics -->

A walrus referenced only by the dropped argument:

```py
"{1}".format(x := 1, x)  # error: [f-string]
```

A dropped argument that calls a function or can raise:

```py
"a".format(foo())  # error: [f-string]
"1".format(1 / 0)  # error: [f-string]
```

## Format-time accessors with side-effecting arguments emit an unsafe fix

`str.format` evaluates every argument before formatting; an f-string interleaves the two. With a
`{[k]}` or `{.attr}` accessor the order is observable, so the fix is unsafe:

<!-- snapshot-diagnostics -->

```py
"{[x]} {}".format(d, len(d))  # error: [f-string]
"{0.attr} {1}".format(obj, side_effect())  # error: [f-string]
```

## Arguments that need parentheses

The argument is wrapped when its bare form would be misparsed: a lambda's colon starts a format spec,
and a leading `{` reads as an escaped brace.

<!-- snapshot-diagnostics -->

```py
"{}".format(lambda: 1)  # error: [f-string]
"{}".format({} | {})  # error: [f-string]
```

## Skipped: rejected by Python or read differently

These parse as `str.format` calls but have no behavior-preserving f-string, so nothing is emitted.

A signed field index (`str.format` raises `KeyError`):

```py
"{+0}".format(0)
```

A non-identifier attribute name (`"{. a}"` calls `getattr(x, " a")`, but `f"{x. a}"` would read
`x.a`):

```py
"{. a}".format(x)
```

A string index whose quote collides with the f-string quote:

```py
"{[']}".format(x)
```

An unknown conversion specifier (`str.format` raises `ValueError`):

```py
"{!?}".format(0)
```

## Drive-by: dropping a leading empty literal leaves no orphan whitespace

<!-- snapshot-diagnostics -->

```py
"" "{}".format(x)  # error: [f-string]
"a" "" "{}".format(x)  # error: [f-string]
```

## Widening: arguments with a non-colliding quote

Rule expansion for [#2031](https://github.com/astral-sh/ruff/issues/2031). Previously any argument
containing a quote was skipped. When the argument's quote differs from the string's own quote, the
interpolation is valid on every supported version, so the conversion is offered:

<!-- snapshot-diagnostics -->

```py
'Magic wand: {}'.format(bag["wand"])  # error: [f-string]
'{}'.format(len(l) * "─")  # error: [f-string]
'Hello {}'.format("world")  # error: [f-string]
```

## Widening: `**locals()` / `**vars()` splats

A `**locals()`, `**vars()`, or `**vars(<target>)` splat resolves keyword fields against the
surrounding scope (or `<target>`'s attributes). The conversion is offered, but the fix is unsafe
because those names are resolved syntactically, with no guarantee they are bound at runtime.

<!-- snapshot-diagnostics -->

`**locals()` and `**vars()` resolve fields to bare names:

```py
"reading {filename}".format(**locals())  # error: [f-string]
"reading {filename}".format(**vars())  # error: [f-string]
```

A `**locals()` splat may be mixed with explicit keyword arguments:

```py
"{prefix}: {filename}".format(prefix='info', **locals())  # error: [f-string]
```

`**vars(<target>)` resolves fields to `<target>.<name>`:

```py
"{foo}:{bar}".format(**vars(x))  # error: [f-string]
"{foo}".format(**vars(self.obj))  # error: [f-string]
```

## PEP 701: same-quote and multi-line interpolations

When the argument reuses the string's own quote, or spans multiple lines, the interpolation is only
valid under [PEP 701](https://peps.python.org/pep-0701/) (Python 3.12+).

### Before Python 3.12

The conversion is not offered, so no diagnostic is emitted:

```toml
target-version = "py311"
lint.select = ["UP032"]
```

```py
"Hello {}".format("world")
"Magic wand: {}".format(bag["wand"])
"{}".format(len(l) * "─")
"{}".format(
    [
        1,
        2,
        3,
    ]
)
"{a}".format(
    a=[
        1,
        2,
        3,
    ]
)
```

### Python 3.12 and later

The same calls are now converted. The single-line cases show the resulting fix. For the multi-line
cases the inline assertion sits inside the call, which suppresses the fix in this snapshot, but the
diagnostic confirms the conversion is now offered:

<!-- snapshot-diagnostics -->

```toml
target-version = "py312"
lint.select = ["UP032"]
```

```py
"Hello {}".format("world")  # error: [f-string]
"Magic wand: {}".format(bag["wand"])  # error: [f-string]
"{}".format(len(l) * "─")  # error: [f-string]
"{}".format(  # error: [f-string]
    [
        1,
        2,
        3,
    ]
)
"{a}".format(  # error: [f-string]
    a=[
        1,
        2,
        3,
    ]
)
```
