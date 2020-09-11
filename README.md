# Groveller

This SRFI should define:

- A simple input language (S-expressions)
- An even simpler output language (Line-oriented S-expressions / LOSE)
- A Scheme procedure to take the input language and write equivalent C program(s).
- The C program, when compiled and run, outputs LOSE.
- The input and output languages should be compatible with Common Lisp.

## The benefits of standardization

Anyone can write a groveller; the trick is to come up with good
interfaces.

- Assume I write a GTK wrapper for the Blub language, and I want to
  use the constant `GTK_STYLE_PROPERTY_BACKGROUND_IMAGE` from the C
  source. The idiomatic Blub constant is
  `GTK::StyleProperty::BackgroundImage` so I write my groveller to emit
  that.
- Another person writes a GTK wrapper for BlubLisp, where the
  idiomatic constant is `gtk:style-property-background-image` so they
  write a groveller to emit that.
- A third person wants to drop the prefixes so they write a groveller
  that emits simply `BackgroundImage`.
- Etc.

These people cannot share the load of wrapping GTK because they are
missing a common language. Each must write a groveller from scratch.

A big part of the work in wrapping constants is simply finding and
listing them all. We can introduce a simple intermediate language just
for writing these listings using the C names:

```
(constant GTK_STYLE_PROPERTY_BACKGROUND_COLOR string)
(constant GTK_STYLE_CLASS_DIM_LABEL string)
(constant GTK_WINDOW_TOPLEVEL unsigned)
```

This input language can be fed to a groveller generator that spits out
a C program. The C program also uses a standard output language:

```
(constant GTK_STYLE_PROPERTY_BACKGROUND_COLOR "background-color")
(constant GTK_STYLE_CLASS_DIM_LABEL "dim-label")
(constant GTK_WINDOW_TOPLEVEL 0)
```

This gives us important benefits:

- The output language is simple and regular, so it can be read easily
  from several different programming languages.
- The output can be saved to a file, and re-used on a different
  machine that has the same C libraries installed but does not have
  their headers.
- Output files can be compared using `diff` to find differences
  between systems.
- Output files can be tracked in version control.
- Output files can be self-describing, saying which OS and
  configuration they were generated on. This makes it possible to ship
  these definitions with a library and match with the underlying OS at
  runtime to find compatible ones.

If we use a simple, regular output language, it can be naturally
exploited for rewriting transformations. Say I use Blub and want to
turn `GTK_STYLE_PROPERTY_BACKGROUND_IMAGE` into
`GTK::StyleProperty::BackgroundImage`. I could have a rewrite rule
just for Blub:

```
(rewrite-constant-prefix GTK_STYLE_PROPERTY_ GTK::StyleProperty:: CamelCase)
```

For BlubLisp we could use:

```
(rewrite-constant-prefix GTK_STYLE_PROPERTY_ gtk:style-property- lisp-case)
```

And so on.

Putting the name mangling in a separate stage of the pipeline means
different languages can re-use the initial groveling stage, its
inputs, and its outputs. Since those are the most burdensome parts,
that's a big win.

## Talking about C types in Lisp

[From SRFI 176]

Each C type name is converted to a symbol as follows:

- `void *` becomes `pointer` and `void (*)()` becomes `function-pointer`
- Any other type `foo *` becomes `foo-pointer`
- Spaces are converted to dashes
- Underscores remain underscores
- Letter case is preserved

For example:

```
unsigned-long-long
unsigned-long-long-pointer
char-pointer-pointer
intmax_t
struct-timespec
struct-timespec-pointer
```

## Input language

```
(pkg-config-path <string>)
(pkg-config <string>)
(include <angle-bracket-header>)
(include "string-header")
(error <string>)
(warning <string>)

(when <expression>
  <body>...)

(type-signedness <type>)
(type-size <type>)
(slot-offset <type> <slot>)
(slot-offset+size <type> <slot>)
(slot-size <type> <slot>)
(constant <constant> signed|unsigned|string)
(call-constant <function> <constant> signed|unsigned|string)
(ifdef-constant <constant> signed|unsigned|string)
(ifdef-call-constant <function> <constant> signed|unsigned|string)
```

### Conditional expressions

```
(not expression)
(and expression ...)
(or expression ...)
(defined identifier)
(= identifier integer)
(< identifier integer)
(> identifier integer)
(<= identifier integer)
(>= identifier integer)
```

## Output language

```
(environment <constant> <value>)
(constant <constant> <value>)
(call-constant <function> <constant> <value>)
(type-signedness <type> <signedness>)
(type-size <type> <size>)
(slot-size <type> <slot> <size>)
(slot-offset <type> <slot> <size>)
```

## Examples

### Input

```
(include <errno.h>)
(ifdef-constant EACCES signed)
(ifdef-call-constant strerror EACCES string)
```

### Output

```
(constant EACCES 13)
(call-constant strerror EACCES "Permission denied")
```
