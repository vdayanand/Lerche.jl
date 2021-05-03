# Grammar Reference

Lerche.jl reads grammars in a syntax developed for the Python Lark
system.  These are referred to as "Lark grammars" below. The following
information is adapted from the grammar reference provided with Lark.

## Definitions

A **grammar** is a list of rules and terminals, that together define a language.

Terminals define the alphabet of the language, while rules define its structure.

In Lerche, a terminal may be a string, a regular expression, or a
concatenation of these and other terminals.

Each rule is a list of terminals and rules, whose location and nesting
define the structure of the resulting parse-tree.

A **parsing algorithm** is an algorithm that takes a grammar
definition and a sequence of symbols (members of the alphabet), and
matches the entirety of the sequence by searching for a structure that
is allowed by the grammar.

### General Syntax and notes

Lark grammars are based on
[EBNF](https://en.wikipedia.org/wiki/Extended_Backus–Naur_form)
syntax, with several enhancements.

EBNF is basically a short-hand for common BNF patterns.

Optionals are expanded:

```ebnf
  a b? c    ->    (a c | a b c)
```

Repetition is extracted into a recursion:

```ebnf
  a: b*    ->    a: _b_tag
                 _b_tag: (_b_tag b)?
```

And so on.

Lark grammars are composed of a list of definitions and directives,
each on its own line. A definition is either a named rule, or a named
terminal, with the following syntax, respectively:

```c
  rule: <EBNF EXPRESSION>
      | etc.

  TERM: <EBNF EXPRESSION>   // Rules aren't allowed
```


**Comments** start with `//` and last to the end of the line (C++ style)

Lerche begins the parse with the rule 'start', unless specified otherwise in the options.

Names of rules are always in lowercase, while names of terminals are
always in uppercase. This distinction has practical effects, for the
shape of the generated parse-tree, and the automatic construction of
the lexer (aka tokenizer, or scanner).


## Terminals

Terminals are used to match text into symbols. They can be defined as
a combination of literals and other terminals.

**Syntax:**

```html
<NAME> [. <priority>] : <literals-and-or-terminals>
```

Terminal names must be uppercase.

Literals can be one of:

* `"string"`
* `/regular expression+/`
* `"case-insensitive string"i`
* `/re with flags/imulx`
* Literal range: `"a".."z"`, `"1".."9"`, etc.

Terminals also support grammar operators, such as `|`, `+`, `*` and `?`.

Terminals are a linear construct, and therefore may not contain
themselves (recursion isn't allowed).

### Templates

Templates are expanded when preprocessing the grammar.

Definition syntax:

```ebnf
  my_template{param1, param2, ...}: <EBNF EXPRESSION>
```

Use syntax:

```ebnf
some_rule: my_template{arg1, arg2, ...}
```

Example:
```ebnf
_separated{x, sep}: x (sep x)*  // Define a sequence of 'x sep x sep x ...'

num_list: "[" _separated{NUMBER, ","} "]"   // Will match "[1, 2, 3]" etc.
```

### Priority

Terminals can be assigned priority only when using a lexer (future
versions may support Earley's dynamic lexing).

Priority can be either positive or negative. If not specified for a terminal, it defaults to 1.

Highest priority terminals are always matched first.

### Regexp Flags

You can use flags on regexps and strings. For example:

```perl
SELECT: "select"i     //# Will ignore case, and match SELECT or Select, etc.
MULTILINE_TEXT: /.+/s
SIGNED_INTEGER: /
    [+-]?  # the sign
    (0|[1-9][0-9]*)  # the digits
 /x
```

Supported flags are one of: `imslux`. See Julia's regex documentation for more details on each one.

Regexps/strings of different flags may not be concatenated.

#### Notes for using a lexer

When using a lexer (standard or contextual), it is the
grammar-author's responsibility to make sure the literals don't
collide, or that if they do, they are matched in the desired
order. Literals are matched according to the following precedence:

1. Highest priority first (priority is specified as: TERM.number: ...)
2. Length of match (for regexps, an estimate of longest theoretical match is used)
3. Length of literal / pattern definition
4. Name

**Examples:**
```perl
IF: "if"
INTEGER : /[0-9]+/
INTEGER2 : ("0".."9")+          //# Same as INTEGER
DECIMAL.2: INTEGER? "." INTEGER  //# Will be matched before INTEGER
WHITESPACE: (" " | /\t/ )+
SQL_SELECT: "select"i
```

### Regular expressions & Ambiguity

Each terminal is eventually compiled to a regular expression. All the
operators and references inside it are mapped to their respective
expressions.

For example, in the following grammar, `A1` and `A2`, are equivalent:
```perl
A1: "a" | "b"
A2: /a|b/
```

This means that inside terminals, Lerche.jl cannot detect or resolve
ambiguity.

For example, for this grammar:
```perl
start           : (A | B)+
A               : "a" | "ab"
B               : "b"
```

We get this behavior:

```bash
>>> parse(p, "ab")
Tree(start, [Token(A, "a"), Token(B, "b")])
```

This is happening because the regex engine always returns the first matching option.

## Rules

**Syntax:**
```html
<name> : <items-to-match>  [-> <alias> ]
       | ...
```

Names of rules and aliases are always in lowercase.

Rule definitions can be extended to the next line by using the OR
operator (signified by a pipe: `|` ).

An alias is a name for the specific rule alternative. It affects tree
construction.


Each item is one of:

* `rule`
* `TERMINAL`
* `"string literal"` or `/regexp literal/`
* `(item item ..)` - Group items
* `[item item ..]` - Maybe. Same as `(item item ..)?`, but when `maybe_placeholders=True`, generates `nothing` if there is no match.
* `item?` - Zero or one instances of item ("maybe")
* `item*` - Zero or more instances of item
* `item+` - One or more instances of item
* `item ~ n` - Exactly *n* instances of item
* `item ~ n..m` - Between *n* to *m* instances of item (not recommended for wide ranges, due to performance issues)

**Examples:**
```perl
hello_world: "hello" "world"
mul: (mul "*")? number     //# Left-recursion is allowed and encouraged!
expr: expr operator expr
    | value               //# Multi-line, belongs to expr

four_words: word ~ 4
```

### Priority

Rules can be assigned priority only when using Earley (future versions
may support LALR as well).

Priority can be either positive or negative. In not specified for a
terminal, it's assumed to be 1 (i.e. the default).

## Directives

### %ignore

All occurrences of the terminal will be ignored, and won't be part of the parse.

Using the `%ignore` directive results in a cleaner grammar.

It's especially important for the LALR(1) algorithm, because adding
whitespace (or comments, or other extraneous elements) explicitly in
the grammar, harms its predictive abilities, which are based on a
lookahead of 1.

**Syntax:**
```html
%ignore <TERMINAL>
```
**Examples:**
```perl
%ignore " "

COMMENT: "#" /[^\n]/*
%ignore COMMENT
```
### %import

Allows one to import terminals and rules from lark grammars.

When importing rules, all their dependencies will be imported into a
namespace, to avoid collisions. It's not possible to override their
dependencies (e.g. like you would when inheriting a class).

**Syntax:**
```html
%import <module>.<TERMINAL>
%import <module>.<rule>
%import <module>.<TERMINAL> -> <NEWTERMINAL>
%import <module>.<rule> -> <newrule>
%import <module> (<TERM1>, <TERM2>, <rule1>, <rule2>)
```

If the module path is absolute, Lerche will attempt to load it from
the built-in directory (which currently contains `common.lark`).

If the module path is relative, such as `.path.to.file`, Lark will
attempt to load it from the current working directory. Grammars must
have the `.lark` extension.

The rule or terminal can be imported under another name with the `->` syntax.

**Example:**
```perl
%import common.NUMBER

%import .terminals_file (A, B, C)

%import .rules_file.rulea -> ruleb
```

Note that `%ignore` directives cannot be imported. Imported rules will
abide by the `%ignore` directives declared in the main grammar.

### %declare

Declare a terminal without defining it. Useful for plugins.
