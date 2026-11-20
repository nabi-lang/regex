# regex

[![CI](https://github.com/nabi-lang/regex/actions/workflows/ci.yml/badge.svg)](https://github.com/nabi-lang/regex/actions/workflows/ci.yml) [![packages.nabi-lang.org](https://img.shields.io/github/v/tag/nabi-lang/regex?label=packages.nabi-lang.org&sort=semver)](https://packages.nabi-lang.org/packages/regex) [![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](#license)

Regular expressions for Nabi, in the style of RE2 and Rust's `regex`:
a search takes time linear in the length of the haystack for every
pattern. Unicode by default (classes, `\w`, `\b`, case folding, UCD
18.0.0), with captures, named groups, iteration, split, replace with
`$name` templates or a function, and `RegexSet` for many patterns in
one pass.

## Add it

```sh
nabi add regex
```

Needs Nabi 0.1 or later.

## Usage

```nabi
use regex::{Captures, Regex};

let re = try Regex::new(r"(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})");
assert(re.is_match("due 2026-11-14"));

let m = re.find("due 2026-11-14") ?? panic("no date");  // byte offsets
println(f"{m.start()}..{m.end()} {m.as_str()}");  // 4..14 2026-11-14

let c = re.captures("due 2026-11-14") ?? panic("no date");
println(f"{c["y"]} {c[2]}");                       // 2026 11

for c in re.captures_iter("2026-01-02, 2025-12-31") {
    println(c.expand("$d.$m.$y"));
}
println(re.replace_all("2026-01-02", "${d}/${m}/${y}"));   // 02/01/2026

let words = try Regex::new(r"\w+");
let caps = words.replace_all("hello wide world", |c: Captures| c[0].to_uppercase());
let parts: List<str> = (try Regex::new(r"\s*[,;]\s*")).split("a, b;c").collect();
```

Offsets are byte offsets and always fall on `char` boundaries.

`regex!` takes a literal pattern, checks it while the program compiles
(a mistake is a compile error with the message `Regex::new` would give)
and compiles it once per thread, at its first use:

```nabi
use regex::regex!;

for line in ["id=7", "name=x"] {
    if regex!(r"^id=\d+$").is_match(line) { println(line); }
}
```

Build options go through `RegexBuilder`:

```nabi
use regex::RegexBuilder;

let re = try RegexBuilder::new(r"^error: (.*)$")
    .case_insensitive(true)
    .multi_line(true)
    .build();
```

and patterns that are searched together through `RegexSet`:

```nabi
let set = try regex::RegexSet::new([r"\d+", r"^GET ", r"\bfail(ed)?\b"]);
println(set.matches("GET /item/42").iter());       // [0, 1]
```

## Syntax

| | |
|---|---|
| `.` | any character but `\n` (`(?s)`: any) |
| `[abc]` `[a-z]` `[^...]` | classes; nest them, `[a-z&&[^aeiou]]`, `--` difference, `~~` symmetric difference |
| `[[:alpha:]]` | ASCII POSIX classes |
| `\d \w \s` `\D \W \S` | Unicode decimal digits, word characters (UTS #18), white space |
| `\pL` `\p{Greek}` `\p{sc=Latin}` `\p{gc=Nd}` `\P{..}` | general categories, scripts, `Alphabetic`, `White_Space`, `Lowercase`, `Uppercase`, `Any`, `ASCII`, `Assigned` |
| `^ $` `\A \z` | line or text start and end (`$` is the end of the text unless `(?m)`) |
| `\b \B` `\< \>` `\b{start} \b{end}` | word boundaries (Unicode) |
| `* + ? {n} {n,} {n,m}` | greedy; add `?` for lazy |
| `a\|b` `(..)` `(?:..)` `(?<name>..)` `(?P<name>..)` | alternation and groups |
| `(?imsxUu)` `(?i:..)` `(?-i)` | case-insensitive, multi-line, `.` matches `\n`, verbose, swap greed, Unicode |
| `\n \t \x7F \x{1F600} é \U0001F600` | escapes; any ASCII punctuation can be escaped |

`regex::escape(text)` makes a pattern that matches `text` itself.

## Linear time

A pattern compiles to a Thompson NFA over UTF-8 bytes. `is_match`,
`find` and the iterators run a lazy DFA: forward for where the match
ends, then backward for where it starts; groups come from a one-pass
DFA when the pattern allows one (each byte leaves one way to go on, as
in `(\d+)-(\w+)`), else from a PikeVM run over just the match. None of
them backtracks: each byte of the haystack is looked at a bounded number
of times, so `(a*)*b` or `(a|aa)*c` on a megabyte of `a` takes as long
as `a*b` does.

Most searches skip most of the haystack without the DFA. A plain
literal is a substring search over the needle's two rarest bytes; a
literal prefix, or the few bytes a match can start with (`(?i)hello`,
`Sherlock|Holmes`, `[Hh]ttp`), takes the DFA straight to the next place
a match can start; a literal in the middle of the pattern (`@` in an
e-mail pattern, ` ERROR ` in a log line pattern) is found first, then
the DFA reads backward to the match start. These scans compile to
vector code.

Memory is bounded too: the compiled program has a size limit (10 MiB by
default, `RegexBuilder::size_limit`; a larger pattern fails with
`RegexError::TooBig`), and each DFA cache has one (2 MiB,
`dfa_size_limit`; a search that keeps overflowing it goes on with the
PikeVM instead of failing).

## Not supported, and why

- **Backreferences** (`\1`, `(?P=name)`, `\k<name>`): matching them is
  NP-hard, so no engine can guarantee linear time with them. Capture the
  group and compare the text in code.
- **Look-around** (`(?=..)`, `(?!..)`, `(?<=..)`, `(?<!..)`): they
  need backtracking or one automaton per assertion. Match the longer
  text and check the condition in code, or run a second regex.
- **Atomic groups and possessive repetition** (`(?>..)`, `a*+`): they
  exist to limit backtracking, which this engine never does.

Each of these is a compile error that says what to write instead, as
are other mistakes (an unclosed group says to add `)`), with the byte
offset and a caret under the pattern.

A `Regex` keeps its search caches inside, so it is not `Sync`; build one
per thread.

## Docs

https://packages.nabi-lang.org/packages/regex/latest/docs

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option. The Unicode tables in `src/tables.nabi` are generated from the
Unicode Character Database, whose terms are in [NOTICE](NOTICE).
