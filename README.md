# ExplicitImports

[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://ericphanson.github.io/ExplicitImports.jl/dev/)
[![Build Status](https://github.com/ericphanson/ExplicitImports.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/ericphanson/ExplicitImports.jl/actions/workflows/CI.yml?query=branch%3Amain)
[![Coverage](https://codecov.io/gh/ericphanson/ExplicitImports.jl/branch/main/graph/badge.svg)](https://codecov.io/gh/ericphanson/ExplicitImports.jl)

## Goal

Figure out what implicit imports a Julia module is relying on, in order to make them explicit.

## Terminology

- _implicit import_: a name `x` available in a module due to `using XYZ` for some package or module `XYZ`. This name has not been explicitly imported; rather, it is simply available since it is exported by `XYZ`.
- _explicit import_: a name `x` available in a module due to `using XYZ: x` or `import XYZ: x` for some package or module `XYZ`.
- _qualified usage_: a name `x` only used in the form `XYZ.x`.

## Why

Relying on implicit imports can be problematic, as Base or another package can start exporting that name as well, resulting in a clash. This is tough because adding a new feature to Base (or a package) and exporting it is not considered a breaking change to its API, but it can cause working code to stop working due to these clashes.

There are various takes on _how problematic_, to what extent this occurs in practice, and to what extent it is worth mitigating. See [julia#42080](https://github.com/JuliaLang/julia/pull/42080) for some discussion on this.

Personally, I don't think this is always a huge issue, and that it's basically fine for packages to use implicit imports if that is their preferred style and they understand the risk. But I do think this issue is somewhat a "hole" in the semver system as it applies to Julia packages, and I wanted to create some tooling to make it easier to mitigate the issue for package authors who would prefer to not rely on implicit imports.

## Implementation status

This seems to be working! However it has not been extensively used or tested.

See the [API docs](https://ericphanson.github.io/ExplicitImports.jl/dev/api/) for the available functionality.

## Example

````julia
julia> using ExplicitImports

julia> print_explicit_imports(ExplicitImports)
WARNING: both JuliaSyntax and Base export "parse"; uses of it in module ExplicitImports must be qualified
Module ExplicitImports is relying on implicit imports for 14 names. These could be explicitly imported as follows:

```julia
using AbstractTrees: Leaves
using AbstractTrees: TreeCursor
using AbstractTrees: children
using AbstractTrees: nodevalue
using DataAPI: outerjoin
using DataFrames: AsTable
using DataFrames: DataFrame
using DataFrames: by
using DataFrames: combine
using DataFrames: groupby
using DataFrames: select!
using DataFrames: subset
using DataFrames: subset!
using Tables: ByRow
```

````

Note: the `WARNING` is more or less harmless; the way this package is written, it will happen any time there is a clash, even if that clash is not realized in your code. I cannot figure out how to suppress it.

## Limitations

### `global` scope quantifier ignored

Currently, my parsing implementation does not take into account the `global` keyword, and thus results may be inaccurate when that is used. This could be fixed by improving the code in `src/get_names_used.jl`.

### Cannot recurse through dynamic `include` statements

These are `include` in which the argument is not a string literal. For example:

```julia
julia> print_explicit_imports(MathOptInterface)
┌ Warning: Dynamic `include` found at /Users/eph/.julia/packages/MathOptInterface/tpiUw/src/Test/Test.jl:631:9; not recursing
└ @ ExplicitImports ~/ExplicitImports/src/get_names_used.jl:37
...
```

In this case, names in files which are included via `include` are not analyzed while parsing.
This can result in inaccurate results, such as false positives in `explicit_imports` and false negatives (or false positives) in `stale_explicit_imports`.

This is essentially an inherent limitation of the approach used in this package. An alternate implementation using an `AbstractInterpreter` (like JET does) may be able to handle this (at the cost of increased complexity).

## Implementation strategy

1. [DONE hackily] Figure out what names used in the module are being used to refer to bindings in global scope (as opposed to e.g. shadowing globals).
   - We do this by parsing the code (thanks to JuliaSyntax), then reimplementing scoping rules on top of the parse tree
   - This is finicky, but assuming scoping doesn't change, should be robust enough (once the long tail of edge cases are dealt with...)
     - Currently, I don't handle the `global` keyword, so those may look like local variables and confuse things
   - This means we need access to the raw source code; `pathof` works well for packages, but for local modules one has to pass the path themselves. Also doesn't seem to work well for stdlibs in the sysimage
2. [DONE] Figure out what implicit imports are available in the module, and which module they come from
    * done, via a magic `ccall` from Discourse, and `Base.which`.
3. [DONE] Figure out which names have been explicitly imported already
   - Done via parsing

Then we can put this information together to figure out what names are actually being used from other modules, and whose usage could be made explicit, and also which existing explicit imports are not being used.
