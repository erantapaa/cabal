% Cabal User Guide: Nix-style local builds

`cabal new-build`, also known as Nix-style local builds, is a new
command inspired by Nix that comes with cabal-install 1.24. Nix-style
local builds combine the best of non-sandboxed and sandboxed Cabal:

1. Like sandboxed Cabal today, we build sets of independent local
   packages deterministically and independent of any global state.
   new-build will never tell you that it can't build your package
   because it would result in a "dangerous reinstall." Given a
   particular state of the Hackage index, your build is completely
   reproducible. For example, you no longer need to compile packages
   with profiling ahead of time; just request profiling and new-build
   will rebuild all its dependencies with profiling automatically.

2. Like non-sandboxed Cabal today, builds of external packages are
   cached in `~/.cabal/store`, so that a package can be built once, and
   then reused anywhere else it is also used. No need to continually
   rebuild dependencies whenever you make a new sandbox: dependencies
   which can be shared, are shared.

Nix-style local builds work with all versions of GHC supported by
cabal-install 1.24, which currently is GHC 7.0 and later.

Some features described in this manual are not implemented.  If you
need them, please give us a shout and we'll prioritize accordingly.

# Quickstart #

Suppose that you are in a directory containing a single Cabal package
which you wish to build.  You can configure and build it using Nix-style
local builds with this command (configuring is not necessary):

~~~~~~~~~~~~~~~~
cabal new-build
~~~~~~~~~~~~~~~~

To open a GHCi shell with this package, use this command:

~~~~~~~~~~~~~~~~
cabal new-repl
~~~~~~~~~~~~~~~~

## Developing multiple packages ##

Many Cabal projects involve multiple packages which need to be built
together.  To build multiple Cabal packages, you need to first create a
`cabal.project` file which declares where all the local package
directories live. For example, in the Cabal repository, there is a root
directory with a folder per package, e.g., the folders `Cabal` and
`cabal-install`. The `cabal.project` file specifies each folder
as part of the project:

~~~~~~~~~~~~~~~~
packages: Cabal/
          cabal-install/
~~~~~~~~~~~~~~~~

The expectation is that a `cabal.project` is checked into your source
control, to be used by all developers of a project.  If you need to
make local changes, they can be placed in `cabal.project.local`
(which should not be checked in.)

Then, to build every component of every package, from the top-level
directory, run the command:  (Warning: cabal-install-1.24 does NOT
have this behavior; you will need to upgrade to HEAD.)

~~~~~~~~~~~~~~~~
cabal new-build
~~~~~~~~~~~~~~~~

To build a specific package, you can either run `new-build` from
the directory of the package in question:

~~~~~~~~~~~~~~~~
cd cabal-install
cabal new-build
~~~~~~~~~~~~~~~~

or you can pass the name of the package as an argument to `cabal
new-build` (this works in any subdirectory of the project):

~~~~~~~~~~~~~~~~
cabal new-build cabal-install
~~~~~~~~~~~~~~~~

You can also specify a specific component of the package to build.
For example, to build a test suite named `package-tests`, use
the command:

~~~~~~~~~~~~~~~~
cabal new-build package-tests
~~~~~~~~~~~~~~~~

Targets can be qualified with package names.  So to request
`package-tests` *from* the `Cabal` package, use `Cabal:package-tests`.

Unlike sandboxes, there is no need to setup a sandbox or `add-source`
projects; just check in `cabal.project` to your repository and
`new-build` will just work.

# How it works #

## Local versus external packages ##

One of the primary innovations of Nix-style local builds is the
distinction between local packages, which users edit and recompile and
must be built per-project, versus external packages, which can
be cached across packages.  To be more precise:

1. A **local package** is one that is listed explicitly in
   the `packages`, `optional-packages` or `extra-packages` field of a
   project.  Usually, these refer to packages whose source code lives
   directly in a folder in your project (although, you can list an
   arbitrary Hackage packages in `extra-packages` to force it to be
   treated as local).

   Local packages, as well as the external packages (below) which depend
   on them, are built **inplace**, meaning that they are always built
   specifically for the project and are not installed globally.  Inplace
   packages are not cached and not given unique hashes, which makes them
   suitable for packages which you want to edit and recompile.

2. A **external package** is any package which is not listed
   in the `packages` field.  The source code for external
   packages is usually retrieved from Hackage.

   When an external package does not depend on an inplace package,
   it can be built and installed to a **global** store, which can be
   shared across projects.  These build products are identified by a
   hash that over all of the inputs which would influence the
   compilation of a package (flags, dependency selection, etc.).  Just
   as in Nix, these hashes uniquely identify the result of a build; if
   we compute this identifier and we find that we already have this ID
   built, we can just use the already built version.

   The global package store is `~/.cabal/store`; if you need to clear
   your store for whatever reason (e.g., to reclaim disk space or
   because the global store is corrupted), deleting this directory is
   safe (`new-build` will just rebuild everything it needs on its next
   invocation).

This split motivates some of the UI choices for Nix-style local
build commands.  For example, flags passed to `cabal new-build` are
only applied to *local* packages, so that adding a flag to `cabal
new-build` doesn't necessitate a rebuild of *every* transitive
dependency in the global package store.

In cabal-install HEAD, Nix-style local builds also take advantage
of a new Cabal library feature,
[per-component builds](https://github.com/ezyang/ghc-proposals/blob/master/proposals/0000-componentized-cabal.rst),
where each component of a package is configured and built separately.
This can massively speed up rebuilds of packages with lots of
components (e.g., a package that defines multiple executables),
as only one executable needs to be rebuilt.  Packages that use
Custom setup scripts are not currently built on a per-component
basis.

## Where are my build products? ##

A major deficiency in the current implementation of new-build is that
there is no programmatic way to access the location of build products.
The location of the build products is intended to be an internal
implementation detail of new-build, but we also understand that
many unimplemented features (e.g., `new-test`) can only be reasonably
worked around by accessing build products directly.

The location where build products can be found varies depending
on the version of cabal-install:

* In cabal-install-1.24, the dist directory for a package `p-0.1`
  is stored in `dist-newstyle/build/p-0.1`.  For example, if you
  built an executable or test suite named `pexe`, it would be located
  at `dist-newstyle/build/p-0.1/build/pexe/pexe`.

* In cabal-install HEAD, the dist directory for a package `p-0.1`
  defining a library built with GHC 8.0.1 on 64-bit Linux
  is `dist-newstyle/build/x86_64-linux/ghc-8.0.1/p-0.1`.
  When per-component builds are enabled (any non-Custom package),
  a subcomponent like an executable or test suite named `pexe` will be
  stored at `dist-newstyle/build/x86_64-linux/ghc-8.0.1/p-0.1/c/pexe`;
  thus, the full path of the executable is
  `dist-newstyle/build/x86_64-linux/ghc-8.0.1/p-0.1/c/pexe/build/pexe/pexe`
  (you can see why we want this to be an implementation detail!)

The paths are a bit longer in HEAD but the benefit is that you can
transparently have multiple builds with different versions of GHC.
We plan to add the ability to create aliases for certain build
configurations, and more convenient paths to access particularly
useful build products like executables.

## Caching ##

Nix-style local builds sport a robust caching system which help
reduce the time it takes to execute a rebuild cycle.  While
the details of how `cabal-install` does caching are an implementation
detail and may change in the future, knowing what gets cached is
helpful for understanding the performance characteristics of
invocations to `new-build`.  The cached intermediate results
are stored in `dist-newstyle/cache`; this folder can be safely
deleted to clear the cache.

The following intermediate results are cached in the following
files in this folder (the most important two are first):

`solver-plan` (binary)
:   The result of calling the dependency solver, assuming that
    the Hackage index, local `cabal.project` file, and
    local `cabal` files are unmodified.  (Notably, we do NOT
    have to dependency solve again if new build products are stored
    in the global store; the invocation of the dependency solver
    is independent of what is already available in the store.)

`source-hashes` (binary)
:   The hashes of all local source files.  When all local source files
    of a local package are unchanged, `cabal new-build` will skip
    invoking `setup build` entirely (saving us from a possibly expensive
    call to `ghc --make`).  The full list of source files participating
    in compilation are determined using `setup sdist --list-sources`
    (thus, if you do not list all your source files in a Cabal file, you
    may fail to recompile when you edit them.)

`config` (same format as `cabal.project`)
:   The full project configuration, merged from `cabal.project` (and
    friends) as well as the command line arguments.

`compiler` (binary)
:   The configuration of the compiler being used to build the project.

`improved-plan` (binary)
:   Like `solver-plan`, but with all non-inplace packages improved into
    pre-existing copies from the store.

Note that every package also has a local cache managed by the
Cabal build system, e.g., in `$distdir/cache`.

There is another useful file in `dist-newstyle/cache`, `plan.json`,
which is a JSON serialization of the computed install plan. (TODO: docs)

# Commands #

We now give an in-depth description of all the commands, describing
the arguments and flags they accept.

## cabal new-configure ##

`cabal new-configure` takes a set of arguments and
writes a `cabal.project.local` file based on the flags passed to this
command.  `cabal new-configure FLAGS; cabal new-build` is roughly equivalent
to `cabal new-build FLAGS`, except that with `new-configure` the flags
are persisted to all subsequent calls to `new-build`.

`cabal new-configure` is intended to be a convenient way to write out a
`cabal.project.local` for simple configurations; e.g., `cabal
new-configure -w ghc-7.8` would ensure that all subsequent builds with
`cabal new-build` are performed with the compiler `ghc-7.8`.  For
more complex configuration, we recommend writing the
`cabal.project.local` file directly (or placing it in `cabal.project`!)

`cabal new-configure` inherits options from `Cabal`.
semantics:

* Any flag accepted by `./Setup configure`.

* Any flag accepted by `cabal configure` beyond `./Setup configure`,
  namely `--cabal-lib-version`, `--constraint`, `--preference` and
  `--solver.`

* Any flag accepted by `cabal install` beyond `./Setup configure`.

* Any flag accepted by `./Setup haddock`.

The options of all of these flags apply only to *local* packages
in a project; this behavior is different than that of `cabal install`,
which applies flags to every package that would be built.  The
motivation for this is to avoid an innocuous addition to the flags
of a package resulting in a rebuild of every package in the store
(which might need to happen if a flag actually applied to every
transitive dependency).  To apply options to an external package,
use a `package` stanza in a `cabal.project` file.

## cabal new-build ##

`cabal new-build` takes a set of targets and builds them.  It
automatically handles building and installing any dependencies
of these targets.

A target can take any of the following forms:

* A package target: `package`, which specifies that all
  enabled components of a package to be built.  By default,
  test suites and benchmarks are *not* enabled, unless
  they are explicitly requested (e.g., via `--enable-tests`.)

* A component target: `[package:][ctype:]component`, which
  specifies a specific component (e.g., a library,
  executable, test suite or benchmark) to be built.

In component targets, `package:` and `ctype:` (valid component
types are `lib`, `exe`, `test` and `bench`) can be used to
disambiguate when multiple packages define the same component,
or the same component name is used in a package (e.g., a package
`foo` defines both an executable and library named `foo`).
We always prefer interpreting a target as a package name rather than as
a component name.

Some example targets:

~~~~
cabal new-build lib:foo-pkg       # build the library named foo-pkg
cabal new-build foo-pkg:foo-tests # build foo-tests in foo-pkg
~~~~

(There is also syntax for specifying module and file targets,
but it doesn't currently do anything.)

Beyond a list of targets, `cabal new-build` accepts all the
flags that `cabal new-configure` takes.  Most of these flags
are only taken into consideration when building local packages;
however, some flags may cause extra store packages to be
built (for example, `--enable-profiling` will automatically
make sure profiling libraries for all transitive dependencies
are built and installed.)

## cabal new-repl ##

`cabal new-repl TARGET` loads all of the modules of the target
into GHCi as interpreted bytecode.  It takes the same flags
as ``cabal new-build``.

Currently, it is not supported to pass multiple targets to ``new-repl``
(``new-repl`` will just successively open a separate GHCi session
for each target.)

## cabal new-freeze ##

`cabal new-freeze` writes out a `cabal.project.freeze` file
which records all of the versions and flags which that are
picked by the solver under the current index and flags.
A `cabal.project.freeze` file has the same syntax as `cabal.project`
and looks something like this::

~~~~
constraints: HTTP ==4000.3.3,
             HTTP +warp-tests -warn-as-error -network23 +network-uri -mtl1 -conduit10,
             QuickCheck ==2.9.1,
             QuickCheck +templatehaskell,
             ...
~~~~

For end-user executables, it is recommended that you distribute the
`cabal.project.freeze` file in your source repository so that
all users see a consistent set of dependencies.  For libraries,
this is not recommended: users often need to build against different
versions of libraries than what you developed against.

## Unsupported commands ##

The following commands are not currently supported:

`cabal new-test` ([#3638](https://github.com/haskell/cabal/issues/3638))
:   Workaround: run the test executable directly (see
    [Where are my build products](#where-are-my-build-products)?)

`cabal new-bench` ([#3638](https://github.com/haskell/cabal/issues/3638))
:   Workaround: run the benchmark executable directly (see
    [Where are my build products](#where-are-my-build-products)?)

`cabal new-run` ([#3638](https://github.com/haskell/cabal/issues/3638))
:   Workaround: run the executable directly (see
    [Where are my build products](#where-are-my-build-products)?)

`cabal new-exec`
:   Workaround: if you wanted to execute GHCi, consider using
    `cabal new-repl` instead.  Otherwise, use `-v` to find the
    list of flags GHC is being invoked with and pass it manually.

`cabal new-haddock` ([#3535](https://github.com/haskell/cabal/issues/3535))
:   Workaround: run `cabal act-as-setup -- haddock --builddir=dist-newstyle/build/pkg-0.1`
    (or execute the Custom setup script directly).

`cabal new-install` ([#3737](https://github.com/haskell/cabal/issues/3737))
:   Workaround: no good workaround at the moment.  (But note that
    you no longer need to install libraries before building!)

# Configuring builds with cabal.project #

`cabal.project` files support a variety of options which configure the
details of your build.  The general syntax of a `cabal.project` file
is similar to that of a Cabal file: there are a number of fields, some
of which live inside stanzas:

~~~~
packages: */*.cabal
with-compiler: /opt/ghc/8.0.1/bin/ghc

package cryptohash
  optimization: False
~~~~

In general, the accepted field names coincide with the accepted
command line flags that `cabal install` and other commands take.
For example, `cabal new-configure --library-profiling` will
write out a project file with `library-profiling: True`.

The full configuration of a project is determined by combining
the following sources (later entries override earlier ones):

1. `~/.cabal/config` (the user-wide global configuration)

2. `cabal.project` (the project configuratoin)

3. `cabal.project.freeze` (the output of `cabal new-freeze`)

4. `cabal.project.local` (the output of `cabal new-configure`)

## Specifying the local packages ##

The following top-level options specify what the local packages of a
project are:

`packages:` _package location list_ (space or comma separated, default: `./*.cabal`)
:   Specifies the list of package locations which contain
    the local packages to be built by this project.  Package locations
    can take the following forms:

    1. They can specify a Cabal file, or a directory containing
       a Cabal file, e.g.,
       `packages: Cabal cabal-install/cabal-install.cabal`

    2. They can specify a glob-style wildcards, which must match one
       or more (a) directories containing a (single) Cabal file, (b) Cabal
       files (extension `.cabal`), or (c) ~~tarballs which
       contain Cabal packages (extension `.tar.gz`)~~ (not implemented yet).  For example, to
       match all Cabal files in all subdirectories, as well as the Cabal
       projects in the parent directories `foo` and `bar`, use
       `packages: */*.cabal ../{foo,bar}/`

    3. ~~They can specify an `http`, `https` or `file` URL, representing
       the path to a remote tarball to be downloaded and built.~~ (not
       implemented yet)

    There is no command line variant of this field; see
    [#3585](https://github.com/haskell/cabal/issues/3585).

`optional-packages:` _package location list_ (space or comma separated, default: `./*/*.cabal`)
:   Like `packages:`, specifies a list of package locations containing
    local packages to be built.  Unlike `packages:`, if we glob for
    a package, it is permissible for the glob to match against zero
    packages.  The intended use-case for `optional-packages` is to make
    it so that vendored packages can be automatically picked up if
    they are placed in a subdirectory, but not error if there aren't
    any.

    There is no command line variant of this field.

`extra-packages:` _package list with version bounds_ (comma separated)
:   ~~Specifies a list of external packages from Hackage which should be
    considered local packages.~~ (Not implemented)

    There is no command line variant of this field.

~~There is also a stanza `source-repository-package` for specifying
packages from an external version control.~~ (Not
implemented.)

All of these options support globs.  `cabal new-build` has
its own glob format:

* Anywhere in a path, as many times as you like,
  you can specify an asterisk `*` wildcard.  E.g., `*/*.cabal` matches
  all `.cabal` files in all immediate subdirectories.  Like
  in glob(7), asterisks do not match hidden files unless there
  is an explicit period, e.g., `.*/foo.cabal` will match
  `.private/foo.cabal` (but `*/foo.cabal` will not).

* You can use braces to specify specific directories; e.g.,
  `{vendor,pkgs}/*.cabal` matches all Cabal files in the
  `vendor` and `pkgs` subdirectories.

Formally, the format described by the following BNF:

~~~~
FilePathGlob    ::= FilePathRoot FilePathGlobRel
FilePathRoot    ::= {- empty -}        # relative to cabal.project
                  | "/"                # Unix root
                  | [a-zA-Z] ":" [/\\] # Windows root
                  | "~"                # home directory
FilePathGlobRel ::= Glob "/"  FilePathGlobRel # Unix directory
                  | Glob "\\" FilePathGlobRel # Windows directory
                  | Glob         # file
                  | {- empty -}  # trailing slash
Glob      ::= GlobPiece *
GlobPiece ::= "*"            # wildcard
            | [^*{},/\\] *   # literal string
            | "\\" [*{},]    # escaped reserved character
            | "{" Glob "," ... "," Glob "}" # union (match any of these)
~~~~

## Global configuration options ##

The following top-level configuration options are not specific
to any package, and thus apply globally:

`verbose:` _nat_ (default: 1)
:   Control the verbosity of `cabal` commands, valid values are
    from 0 to 3.

    The command line variant of this field is `--verbose=2`;
    a short form `-v2` is also supported.

`jobs:` _nat_ or `$ncpus` (default: 1)
:   Run _nat_ jobs simultaneously when building.  If `$ncpus` is
    specified, run the number of jobs equal to the number of CPUs.
    Package building is often quite parallel, so turning on parallelism
    can speed up build times quite a bit!

    The command line variant of this field is `--jobs=2`;
    a short form `-j2` is also supported; a bare `--jobs` or
    `-j` is equivalent to `--jobs=$ncpus`.

`keep-going:` _boolean_ (default: False)
:   If true, after a build failure, continue to build other unaffected
    packages.

    The command line variant of this field is `--keep-going`.

## Solver configuration options ##

The following settings control the behavior of the dependency
solver:

`constraints:` _constraints_ (comma separated)
:   Add extra constraints to the version bounds, flag settings,
    and other properties a solver can pick for a package.  For example,
    to only consider install plans that do not use `bar` at all, or use
    `bar-2.1`, write:

    ~~~~~~~~~~~~~~~~
    constraints: bar == 2.1
    ~~~~~~~~~~~~~~~~

    Version bounds have the same syntax as `build-depends`.
    You can also specify flag assignments:

    ~~~~~~~~~~~~~~~~
    # Require bar to be installed with the foo flag turned on and
    # the baz flag turned off
    constraints: bar +foo -baz

    # Require that bar NOT be present in the install plan. Note:
    # this is just syntax sugar for '> 1 && < 1', and is supported
    # by build-depends.
    constraints: bar -none
    ~~~~~~~~~~~~~~~~

    A package can be specified multiple times in `constraints`, in which
    case the specified constraints are intersected.  This is useful,
    since the syntax does not allow you to specify multiple constraints
    at once.  For example, to specify both version bounds and flag
    assignments, you would write:

    ~~~~~~~~~~~~~~~~
    constraints: bar == 2.1,
                 bar +foo -baz,
    ~~~~~~~~~~~~~~~~

    There are also some more specialized constraints, which most people
    don't generally need:

    ~~~~~~~~~~~~~~~~
    # Require bar to be preinstalled in the global package database
    # (this does NOT include the Nix-local build global store.)
    constraints: bar installed

    # Require the local source copy of bar to be used
    # (Note: By default, if we have a local package we will
    # automatically use it, so it generally not be necessary to
    # specify this)
    constraints: bar source

    # Require that bar be solved with test suites and benchmarks enabled
    # (Note: By default, new-build configures the solver to make
    # a best-effort attempt to enable these stanzas, so this generally
    # should not be necessary.)
    constraints: bar test,
                 bar bench
    ~~~~~~~~~~~~~~~~

    The command line variant of this field is `--constraint="pkg >= 2.0"`; to specify
    multiple constraints, pass the flag multiple times.

`preferences:` _preference_ (comma separated)
:   Like `constraints`, but the solver will attempt to satisfy these
    preferences on a best-effort basis.  The resulting install is
    locally optimal with respect to preferences; specifically,
    no single package could be replaced with a more preferred version
    that still satisfies the hard constraints.

    Operationally, preferences can cause the solver to attempt certain
    version choices of a package before others, which can improve
    dependency solver runtime.

    One way to use `preferences` is to take a known working set of
    constraints (e.g., via `cabal new-freeze`) and record them as
    preferences. In this case, the solver will first attempt to use this
    configuration, and if this violates hard constraints, it will try to
    find the minimal number of upgrades to satisfy the hard constraints
    again.

    The command line variant of this field is `--preference="pkg >= 2.0"`; to specify
    multiple preferences, pass the flag multiple times.

`allow-newer:` `none` _or_ `all` _or_ _list of scoped package names_ (space or comma separated, default: `none`)
:   Allow the solver to pick an newer version of some packages than
    would normally be permitted by than the `build-depends` bounds
    of packages in the install plan.  This option may be useful if
    the dependency solver cannot otherwise find a valid install plan.

    For example, to relax `pkg`s `build-depends` upper bound on
    `dep-pkg`, write a scoped package name of the form:

    ~~~~~~~~~~~~~~~~
    allow-newer: pkg:dep-pkg
    ~~~~~~~~~~~~~~~~

    This syntax is recommended, as it is often only a single package
    whose upper bound is misbehaving.  In this case, the upper bounds of
    other packages should still be respected; indeed, relaxing the bound
    can break some packages which test the selected version of packages.

    However, in some situations (e.g., when attempting to build
    packages on a new version of GHC), it is useful to disregard
    *all* upper-bounds, with respect to a package or all packages.
    This can be done by specifying just a package name, or using
    the keyword `all` to specify all packages:

    ~~~~~~~~~~~~~~~~
    # Disregard upper bounds involving the dependencies on
    # packages bar, baz and quux
    allow-newer: bar, baz, quux

    # Disregard all upper bounds when dependency solving
    allow-newer: all
    ~~~~~~~~~~~~~~~~

    `allow-newer` is often used in conjunction with a constraint
    (in the `constraints` field) forcing the usage of a specific,
    newer version of a package.

    The command line variant of this field is `--allow-newer=bar`.
    A bare `--allow-newer` is equivalent to `--allow-newer=all`.

`allow-older:` `none` _or_ `all` _or_ _list of scoped package names_ (space or comma separated, default: `none`)
:   Like `allow-newer`, but applied to lower bounds rather than upper
    bounds.

    The command line variant of this field is `--allow-older=all`.
    A bare `--allow-older` is equivalent to `--allow-older=all`.

## Package configuration options ##

Package options affect the building of specific packages.  There
are two ways a package option can be specified:

* They can be specified at the top-level, in which case they
  apply only to **local package**, or

* They can be specified inside a `package` stanza, in which
  case they apply to the build of the package, whether or not
  it is local or external.

For example, the following options specify that `optimization`
should be turned off for all local packages, and that `bytestring`
(possibly an external dependency) should be built with
`-fno-state-hack`:

~~~~~~~~~~~~~~~~
optimization: False

package bytestring
    ghc-options: -fno-state-hack
~~~~~~~~~~~~~~~~

At the moment, there is no way to specify an option to apply
to all external packages or all inplace packages.  Additionally,
it is only possible to specify these options on the command
line for all local packages (there is no per-package command
line interface.)

Some flags were added by more recent versions of the Cabal
library.  This means that they are NOT supported by packages
which use Custom setup scripts that require a version of the
Cabal library older than when the feature was added.

`flags:` _list of +flagname or -flagname_ (space separated)
:   Force all flags specified as `+flagname` to be true, and
    all flags specified as `-flagname` to be false.  For
    example, to enable the flag `foo` and disable `bar`, set:

    ~~~~~~~~~~~~~~~~
    flags: +foo -bar
    ~~~~~~~~~~~~~~~~

    If there is no leading punctuation, it is assumed that the
    flag should be enabled; e.g., this is equivalent:

    ~~~~~~~~~~~~~~~~
    flags: foo -bar
    ~~~~~~~~~~~~~~~~

    Flags are *per-package*, so it doesn't make much sense to specify
    flags at the top-level, unless you happen to know that *all* of your
    local packages support the same named flags.  If a flag is
    not supported by a package, it is ignored.

    See also the solver configuration field `constraints`.

    The command line variant of this flag is `--flags`.  There
    is also a shortened form `-ffoo -f-bar`.

    A common mistake is to say `cabal new-build -fhans`, where
    `hans` is a flag for a transitive dependency that is not in
    the local package; in this case, the flag will be silently ignored.
    If `haskell-tor` is the package you want this flag to apply
    to, try `--constraint="haskell-tor +hans"` instead.

`with-compiler:` _executable_
:   Specify the path to a particular compiler to be used.  If not an
    absolute path, it will be resolved according to the `PATH`
    environment.  The type of the compiler (GHC, GHCJS, etc)
    must be consistent with the setting of the `compiler` field.

    The most common use of this option is to specify a different
    version of your compiler to be used; e.g., if you have `ghc-7.8`
    in your path, you can specify `with-compiler: ghc-7.8` to use it.

    This flag also sets the default value of `with-hc-pkg`, using
    the heuristic that it is named `ghc-pkg-7.8` (if your executable
    name is suffixed with a version number), or is the executable
    named `ghc-pkg` in the same directory as the `ghc` directory.
    If this heuristic does not work, set `with-hc-pkg` explicitly.

    For inplace packages, `cabal new-build` maintains a separate
    build directory for each version of GHC, so you can maintain
    multiple build trees for different versions of GHC without
    clobbering each other.

    At the moment, it's not possible to set `with-compiler` on
    a per-package basis, but eventually we plan on relaxing
    this restriction.  If this is something you need, give
    us a shout.

    The command line variant of this flag is `--with-compiler=ghc-7.8`;
    there is also a short version `-w ghc-7.8`.

`with-hc-pkg:` _executable_
:   Specify the path to the package tool, e.g., `ghc-pkg`.  This
    package tool must be compatible with the compiler specified by
    `with-compiler` (generally speaking, it should be precisely
    the tool that was distributed with the compiler). If this option is
    omitted, the default value is determined from `with-compiler`.

    The command line variant of this flag is `--with-hc-pkg=ghc-pkg-7.8`.

`optimization:` _nat_ (default: `1`)
:   Build with optimization. This is appropriate for production use,
    taking more time to build faster libraries and programs.

    The optional _nat_ value is the optimisation level. Some compilers
    support multiple optimisation levels. The range is 0 to 2. Level 0
    disables optimization, level 1 is the default. Level 2 is higher
    optimisation if the compiler supports it. Level 2 is likely to lead
    to longer compile times and bigger generated code.  If you are
    not planning to run code, turning off optimization will lead
    to better build times and less code to be rebuilt when a module
    changes.

    We also accept `True` (equivalent to 1) and `False` (equivalent to
    0).

    Note that as of GHC 8.0, GHC does not recompile when
    optimization levels change (see [#10923](https://ghc.haskell.org/trac/ghc/ticket/10923)),
    so if you change the optimization level for a local package you may
    need to blow away your old build products in order to rebuild
    with the new optimization level.

    The command line variant of this flag is `-O2` (with `-O1` equivalent to `-O`).
    There are also long-form variants `--enable-optimization` and
    `--disable-optimization`.

`configure-options:` _args_ (space separated)
:   A list of extra arguments to pass to the external `./configure` script,
    if one is used.  This is only useful for packages which have the
    `Configure` build type. See also the section on [system-dependent
    parameters](developing-packages.html#system-dependent-parameters).

    The command line variant of this flag is `--configure-option=arg`,
    which can be specified multiple times to pass multiple options.

`compiler:` `ghc` _or_ `ghcjs` _or_ `jhc` _or_ `lhc` _or_ `uhc` _or_ `haskell-suite` (default: `ghc`)
:   Specify which compiler toolchain to be used.  This is independent
    of `with-compiler`, because the choice of toolchain affects
    Cabal's build logic.

    The command line variant of this flag is `--compiler=ghc`.

`tests:` _boolean_ (default: `False`)
:   Force test suites to be enabled.  For most users
    this should not be needed, as we always attempt to solve for test
    suite dependencies, even when this value is `False`; furthermore,
    test suites are automatically enabled if they are requested as a
    built target.

    The command line variant of this flag is `--enable-tests`
    and `--disable-tests`.

`benchmarks:` _boolean_ (default: `False`)
:   Force benchmarks to be enabled.  For most users
    this should not be needed, as we always attempt to solve for
    benchmark dependencies, even when this value is `False`;
    furthermore, benchmarks are automatically enabled if they are
    requested as a built target.

    The command line variant of this flag is `--enable-benchmarks`
    and `--disable-benchmarks`.

`extra-prog-path:` _paths_ (newline or comma separated, added in Cabal 1.18)
:   A list of directories to search for extra required programs.
    Most users should not need this, as programs like `happy`
    and `alex` will automatically be installed and added to
    the path.  This can be useful if a `Custom` setup script
    relies on an exotic extra program.

    The command line variant of this flag is `--extra-prog-path=PATH`,
    which can be specified multiple times.

`run-tests:` _boolean_ (default: `False`)
:   Run the package test suite upon installation.  This is useful
    for saying "When this package is installed, check that the test
    suite passes, terminating the rest of the build if it is broken."

    One deficiency: the `run-test` setting of a package is
    NOT recorded as part of the hash, so if you install something
    without `run-tests` and then turn on `run-tests`, we won't
    subsequently test the package.  If this is causing you problems,
    give us a shout.

    The command line variant of this flag is `--run-tests`.

### Object code options ###

`debug-info:` _boolean_ (default: False, added in Cabal 1.22)
:   If the compiler (e.g., GHC 7.10 and later) supports outputing
    OS native debug info (e.g., DWARF), setting `debug-info: True` will
    instruct it to do so.  See the GHC wiki page on
    [DWARF](https://ghc.haskell.org/trac/ghc/wiki/DWARF) for more
    information about this feature.

    (This field also accepts numeric syntax, but as of GHC 8.0
    this doesn't do anything.)

    The command line variant of this flag is `--enable-debug-info`
    and `--disable-debug-info`.

`split-objs:` _boolean_ (default: False)
:   Use the GHC `-split-objs` feature when building the library. This
    reduces the final size of the executables that use the library by
    allowing them to link with only the bits that they use rather than
    the entire library. The downside is that building the library takes
    longer and uses considerably more memory.

    The command line variant of this flag is `--enable-split-objs`
    and `--disable-split-objs`.

`executable-stripping:` _boolean_ (default: True)
:   When installing binary executable programs, run the
    `strip` program on the binary. This can considerably reduce the size
    of the executable binary file. It does this by removing debugging
    information and symbols.

    Not all Haskell implementations generate native binaries. For such
    implementations this option has no effect.

    (TODO: Check what happens if you combine this with `debug-info`.)

    The command line variant of this flag is `--enable-executable-stripping`
    and `--disable-executable-stripping`.

`library-stripping:` _boolean_ (added in Cabal 1.19)
:   When installing binary libraries, run the `strip` program on the
    binary, saving space on the file system.  See also `executable-stripping`.

    The command line variant of this flag is `--enable-library-stripping`
    and `--disable-library-stripping`.

### Executable options ###

`program-prefix:` _prefix_
:   ~~Prepend _prefix_ to installed program names.~~ (Currently implemented
    in a silly and not useful way. If you need this to work give us a
    shout.)

    _prefix_ may contain the following path variables: `$pkgid`, `$pkg`,
    `$version`, `$compiler`, `$os`, `$arch`, `$abi`, `$abitag`

    The command line variant of this flag is `--program-prefix=foo-`.

`program-suffix:` _suffix_
:   ~~Append _suffix_ to installed program names.~~ (Currently implemented
    in a silly and not useful way. If you need this to work give us a
    shout.)

    The most obvious use for this is to append the program's version
    number to make it possible to install several versions of a program
    at once: `program-suffix: $version`.

    _suffix_ may contain the following path variables: `$pkgid`, `$pkg`,
    `$version`, `$compiler`, `$os`, `$arch`, `$abi`, `$abitag`

    The command line variant of this flag is `--program-suffix='$version'`.

### Dynamic linking options ###

`shared:` _boolean_ (default: False)
:   Build shared library. This implies a separate compiler run to
    generate position independent code as required on most platforms.

    The command line variant of this flag is `--enable-shared`
    and `--disable-shared`.

`executable-dynamic:` _boolean_ (default: False)
:   Link executables dynamically. The executable's library dependencies should
    be built as shared objects. This implies `shared: True` unless
    `shared: False` is explicitly specified.

    The command line variant of this flag is `--enable-executable-dynamic`
    and `--disable-executable-dynamic`.

`library-for-ghci:` _boolean_ (default: True)
:   Build libraries suitable for use with GHCi.  This involves an extra
    linking step after the build.

    Not all platforms support GHCi and indeed on some platforms, trying
    to build GHCi libs fails. In such cases, consider setting
    `library-for-ghci: False`.

    The command line variant of this flag is `--enable-library-for-ghci`
    and `--disable-library-for-ghci`.

`relocatable:` (default: False, added in Cabal 1.21)
:   ~~Build a package which is relocatable.~~
    (TODO: It is not clear what this actually does, or if it works at all.)

    The command line variant of this flag is `--relocatable`.

### Foreign function interface options ###

`extra-include-dirs:` _directories_ (comma or newline separated list)
:   An extra directory to search for C header files. You can use this
    flag multiple times to get a list of directories.

    You might need to use this flag if you have standard system header
    files in a non-standard location that is not mentioned in the
    package's `.cabal` file. Using this option has the same affect as
    appending the directory _dir_ to the `include-dirs` field in each
    library and executable in the package's `.cabal` file. The advantage
    of course is that you do not have to modify the package at all.
    These extra directories will be used while building the package and
    for libraries it is also saved in the package registration
    information and used when compiling modules that use the library.

    The command line variant of this flag is `--extra-include-dirs=DIR`,
    which can be specified multiple times.

`extra-lib-dirs:` _directories_ (comma or newline separated list)
:   An extra directory to search for system libraries files.

    The command line variant of this flag is `--extra-lib-dirs=DIR`,
    which can be specified multiple times.

`extra-framework-dirs:` _directories_ (comma or newline separated list)
:   An extra directory to search for frameworks (OS X only).

    You might need to use this flag if you have standard system
    libraries in a non-standard location that is not mentioned in the
    package's `.cabal` file. Using this option has the same affect as
    appending the directory _dir_ to the `extra-lib-dirs` field in each
    library and executable in the package's `.cabal` file. The advantage
    of course is that you do not have to modify the package at all.
    These extra directories will be used while building the package and
    for libraries it is also saved in the package registration
    information and used when compiling modules that use the library.

    The command line variant of this flag is `--extra-framework-dirs=DIR`,
    which can be specified multiple times.

### Profiling options ###

`profiling:` _boolean_ (default: False, added in Cabal 1.21)
:   Build libraries and executables with profiling enabled (for compilers
    that support profiling as a separate mode).  It is only necessary
    to specify `profiling` for the specific package you want to profile;
    `cabal new-build` will ensure that all of its transitive
    dependencies are built with profiling enabled.

    To enable profiling for only libraries or executables, see
    `library-profiling` and `executable-profiling`.

    For useful profiling, it can be important to control precisely what
    cost centers are allocated; see `profiling-detail`.

    The command line variant of this flag is `--enable-profiling` and
    `--disable-profiling`.

`library-vanilla:` _boolean_ (default: True)
:   Build ordinary libraries (as opposed to profiling libraries).
    Mostly, you can set this to False to avoid building ordinary
    libraries when you are profiling.

    The command line variant of this flag is `--enable-library-vanilla`
    and `--disable-library-vanilla`.

`library-profiling:` _boolean_ (default: False, added in Cabal 1.21)
:   Build libraries with profiling enabled.

    The command line variant of this flag is `--enable-library-profiling`
    and `--disable-library-profiling`.

`executable-profiling:` _boolean_ (default: False, added in Cabal 1.21)
:   Build executables with profiling enabled.

    The command line variant of this flag is `--enable-executable-profiling`
    and `--disable-executable-profiling`.

`profiling-detail:` _level_ (added in Cabal 1.23)
:   Some compilers that support profiling, notably GHC, can allocate costs to
    different parts of the program and there are different levels of
    granularity or detail with which this can be done. In particular for GHC
    this concept is called "cost centers", and GHC can automatically add cost
    centers, and can do so in different ways.

    This flag covers both libraries and executables, but can be overridden
    by the `library-profiling-detail` field.

    Currently this setting is ignored for compilers other than GHC. The levels
    that cabal currently supports are:

    `default`
    :    For GHC this uses `exported-functions` for libraries and
         `toplevel-functions` for executables.

    `none`
    :    No costs will be assigned to any code within this component.

    `exported-functions`
    :    Costs will be assigned at the granularity of all top level functions
         exported from each module. In GHC specifically, this is for non-inline
         functions.

    `toplevel-functions`
    :    Costs will be assigned at the granularity of all top level functions
         in each module, whether they are exported from the module or not.
         In GHC specifically, this is for non-inline functions.

    `all-functions`
    :    Costs will be assigned at the granularity of all functions in each
         module, whether top level or local. In GHC specifically, this is for
         non-inline toplevel or where-bound functions or values.

    The command line variant of this flag is `--profiling-detail=none`.

`library-profiling-detail:` _level_ (added in Cabal 1.23)
:   Like `profiling-detail`, but applied only to libraries

    The command line variant of this flag is `--library-profiling-detail=none`.

### Coverage options ###

`coverage:` _boolean_ (default: False, added in Cabal 1.21)
:   Build libraries and executables (including test suites) with Haskell
    Program Coverage enabled. Running the test suites will automatically
    generate coverage reports with HPC.

    The command line variant of this flag is `--enable-coverage`
    and `--disable-coverage`.

`library-coverage:` _boolean_ (default: False, added in Cabal 1.21)
:   Deprecated, use `coverage`.

    The command line variant of this flag is `--enable-library-coverage`
    and `--disable-library-coverage`.

### Haddock options ###

Documentation building support is fairly sparse at the moment.
Let us know if it's a priority for you!

`documentation:` _boolean_ (default: False)
:   Enables building of Haddock documentation

    The command line variant of this flag is `--enable-documentation`
    and `--disable-documentation`.

`doc-index-file`: _templated path_
:   A central index of Haddock API documentation (template cannot use
    `$pkgid`), which should be updated as documentation is built.

    The command line variant of this flag is `--doc-index-file=TEMPLATE`

The following commands are equivalent to ones that would be passed
when running `setup haddock`.  (TODO: Where does the documentation
get put.)

`haddock-hoogle:` _boolean_ (default: False)
:   Generate a text file which can be
    converted by [Hoogle](http://www.haskell.org/hoogle/) into a
    database for searching. This is equivalent to running `haddock`
    with the `--hoogle` flag.

    The command line variant of this flag is `--hoogle` (for the `haddock` command).

`haddock-html:` _boolean_ (default: True)
:   Build HTML documentation.

    The command line variant of this flag is `--html` (for the `haddock` command).

`haddock-html-location:` _templated path_
:   Specify a template for the location of HTML documentation for
    prerequisite packages.  The substitutions are applied to the
    template to obtain a location for each package, which will be used
    by hyperlinks in the generated documentation. For example, the
    following command generates links pointing at [Hackage] pages:

    ~~~~~~~~~~~~~~~~
    html-location: 'http://hackage.haskell.org/packages/archive/$pkg/latest/doc/html'
    ~~~~~~~~~~~~~~~~

    Here the argument is quoted to prevent substitution by the shell. If
    this option is omitted, the location for each package is obtained
    using the package tool (e.g. `ghc-pkg`).

    The command line variant of this flag is `--html-location` (for the
    `haddock` subcommand).

`haddock-executables:` _boolean_ (default: False)
:   Run haddock on all executable programs.

    The command line variant of this flag is `--executables` (for the
    `haddock` subcommand).

`haddock-tests:` _boolean_ (default: False)
:   Run haddock on all test suites.

    The command line variant of this flag is `--tests` (for the
    `haddock` subcommand).

`haddock-benchmarks:` _boolean_ (default: False)
:   Run haddock on all benchmarks.

    The command line variant of this flag is `--benchmarks` (for the
    `haddock` subcommand).

`haddock-all:` _boolean_ (default: False)
:   Run haddock on all components.

    The command line variant of this flag is `--all` (for the
    `haddock` subcommand).

`haddock-internal:` _boolean_ (default: False)
:   Build haddock documentation which includes unexposed modules and symbols.

    The command line variant of this flag is `--internal` (for the
    `haddock` subcommand).

`haddock-css:` _path_
:   The CSS file that should be used to style the generated
    documentation (overriding haddock's default.)

    The command line variant of this flag is `--css` (for the
    `haddock` subcommand).

`haddock-hyperlink-source:` _boolean_ (default: False)
:   Generated hyperlinked source code using `HsColour`, and have
    Haddock documentation link to it.

    The command line variant of this flag is `--hyperlink-source` (for the
    `haddock` subcommand).

`haddock-hscolour-css:` _path_
:   The CSS file that should be used to style the generated hyperlinked
    source code (from `HsColour`).

    The command line variant of this flag is `--hscolour-css` (for the
    `haddock` subcommand).

`haddock-contents-location:` _url_
:   A baked-in URL to be used as the location for the contents page.

    The command line variant of this flag is `--contents-location` (for the
    `haddock` subcommand).

`haddock-keep-temp-files:`
:   Keep temporary files.

    The command line variant of this flag is `--keep-temp-files` (for the
    `haddock` subcommand).

## Advanced global configuration options ##

`http-transport:` `curl` or `wget` or `powershell` or `plain-http` (default: `curl`)
:   Set a transport to be used when making http(s) requests.

    The command line variant of this field is `--http-transport=curl`.

`ignore-expiry:` _boolean_ (default: False)
:   If `True`, we will ignore expiry dates on metadata from Hackage.

    In general, you should not set this to `True` as it will leave
    you vulnerable to stale cache attacks.  However, it
    may be temporarily useful if the main Hackage server is
    down, and we need to rely on mirrors which have not been updated
    for longer than the expiry period on the timestamp.

    The command line variant of this field is `--ignore-expiry`.

`remote-repo-cache:` _directory_ (default: `~/.cabal/packages`)
:   ~~The location where packages downloaded from remote repositories
    will be cached.~~ Not implemented yet.

    The command line variant of this flag is `--remote-repo-cache=DIR`.

`logs-dir:` _directory_ (default: `~/.cabal/logs`)
:   ~~The location where build logs for packages are stored.~~
    Not implemented yet.

    The command line variant of this flag is `--logs-dir=DIR`.

`build-summary:` _template filepath_ (default: `~/.cabal/logs/build.log`)
:   ~~The file to save build summaries.  Valid variables which
    can be used in the path are `$pkgid`, `$compiler`, `$os` and
    `$arch`.~~ Not implemented yet.

    The command line variant of this flag is `--build-summary=TEMPLATE`.

`local-repo:` _directory_
:   ~~The location of a local repository.~~ Deprecated. See "Legacy repositories."

    The command line variant of this flag is `--local-repo=DIR`.

`world-file:` _path_
:   ~~The location of the world file.~~ Deprecated.

    The command line variant of this flag is `--world-file=FILE`.

Undocumented fields: `root-cmd`, `symlink-bindir`,
`build-log`, `remote-build-reporting`, `report-planned-failure`,
`one-shot`, `offline`.

### Advanced solver options ###

Most users generally won't need these.

`solver:` `modular`
:   This field is reserved to allow the specification of alternative
    dependency solvers.  At the moment, the only accepted option
    is `modular`.

    The command line variant of this field is `--solver=modular`.

`max-backjumps:` _nat_ (default: 2000)
:   Maximum number of backjumps (backtracking multiple steps) allowed
    while solving.  Set -1 to allow unlimited backtracking, and 0 to
    disable backtracking completely.

    The command line variant of this field is `--max-backjumps=2000`.

`reorder-goals:` _boolean_ (default: False)
:   When enabled, the solver will reorder goals according to
    certain heuristics.  Slows things down on average,
    but may make backtracking faster for some packages.
    It's unlikely to help for small projects, but for big
    install plans it may help you find a plan when otherwise
    this is not possible.  See [#1780](https://github.com/haskell/cabal/issues/1780)
    for more commentary.

    The command line variant of this field is `--(no-)reorder-goals`.

`count-conflicts:` _boolean_ (default: True)
:   Try to speed up solving by preferring goals that are involved in a
    lot of conflicts.

    The command line variant of this field is `--(no-)count-conflicts`.

`strong-flags:` _boolean_ (default: False)
:   Do not defer flag choices. (TODO: Better documentation.)

    The command line variant of this field is `--(no-)strong-flags`.

`cabal-lib-version:` _version_
:   This field selects the version of the Cabal library which should
    be used to build packages.  This option is intended primarily
    for internal development use (e.g., forcing a package to build
    with a newer version of Cabal, to test a new version of Cabal.)
    (TODO: Specify its semantics more clearly.)

    The command line variant of this field is `--cabal-lib-version=1.24.0.1`.
