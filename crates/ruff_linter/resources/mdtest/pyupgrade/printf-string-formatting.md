# `printf-string-formatting` (`UP031`)

Rule expansion for [#2031](https://github.com/astral-sh/ruff/issues/2031). Previously, UP031 silently
dropped several right-hand-side expression kinds (binary operations, unary operations, and similar)
when rewriting `%` formatting. They are now wrapped and preserved, which also lets them flow through
the UP031 to UP032 chain.

```toml
lint.select = ["UP031"]
```

## Binary and unary operations on the right-hand side

The right-hand side is parenthesized so its value is forwarded to `str.format` unchanged. The fix is
unsafe because `%` formatting and `str.format` can disagree on edge cases (for example, a single
right-hand-side tuple):

<!-- snapshot-diagnostics -->

```py
"├─%s─" % (len(l) * "─")  # error: [printf-string-formatting]
"%s" % ("a" + "b")  # error: [printf-string-formatting]
"%s" % (-x)  # error: [printf-string-formatting]
```

## Mapping built from a call

A mapping right-hand side is forwarded with `**`, so keyed conversions keep resolving by name. This
also lets the rewritten `.format()` call flow through the UP031 to UP032 chain when UP032 is enabled:

<!-- snapshot-diagnostics -->

```py
"%(foo)s" % vars(obj)  # error: [printf-string-formatting]
```
