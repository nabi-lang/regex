# Changelog

All notable changes to this package. Versions follow semver, independent of the Nabi toolchain.

## 0.1.0 (unreleased)

First release.

- `Regex` with search time linear in the haystack for every pattern: lazy
  DFA, one-pass DFA and PikeVM over a Thompson NFA, no backtracking.
- Unicode by default: classes, scripts, `\w`, `\b`, case folding (UCD
  18.0.0); nested classes with intersection and difference.
- Captures, named groups, iteration, `split`, and replace with `$name`
  templates or a function.
- `regex!` checks a literal pattern at compile time; `RegexBuilder` for
  options; `RegexSet` for many patterns in one pass; `escape`.
- Literal prefilters skip most of the haystack before the DFA runs.
- No backreferences, look-around or atomic groups: each is a compile error
  that says what to write instead.
- Needs Nabi 0.1 or later.
