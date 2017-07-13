# Weeder [![Hackage version](https://img.shields.io/hackage/v/weeder.svg?label=Hackage)](https://hackage.haskell.org/package/weeder) [![Stackage version](https://www.stackage.org/package/weeder/badge/lts?label=Stackage)](https://www.stackage.org/package/weeder) [![Linux Build Status](https://img.shields.io/travis/ndmitchell/weeder.svg?label=Linux%20build)](https://travis-ci.org/ndmitchell/weeder) [![Windows Build Status](https://img.shields.io/appveyor/ci/ndmitchell/weeder.svg?label=Windows%20build)](https://ci.appveyor.com/project/ndmitchell/weeder)

Most projects accumulate code over time. Weeder detects unused Haskell exports, allowing dead code to be removed (pulling up the weeds). Weeder piggy-backs off files generated by [`stack`](https://www.haskellstack.org), so first obtain stack, then:

* Install `weeder` by running `stack install weeder --resolver=nightly`.
* Ensure your project has a `stack.yaml` file. If you don't normally build with `stack` then run `stack init` to generate one.
* Run `weeder . --build`, which builds your project with `stack` and reports any weeds.

## What does Weeder detect?

Weeder detects a bunch of weeds, including:

* You export a function `helper` from module `Foo.Bar`, but nothing else in your package uses `helper`, and `Foo.Bar` is not an `exposed-module`. Therefore, the export of `helper` is a weed. Note that `helper` itself may or may not be a weed - once it is no longer exported `-fwarn-unused-binds` will tell you if it is entirely redundant.
* Your package `depends` on another package but doesn't use anything from it - the dependency should usually be deleted. This functionality is quite like [packunused](https://hackage.haskell.org/package/packunused), but implemented quite differently.
* Your package has entries in the `other-modules` field that are either unused (and thus should be deleted), or are missing (and thus should be added). The `stack` tool warns about the latter already.
* A source file is used between two different sections in a `.cabal` file - e.g. in both the library and the executable. Usually it's better to arrange for the executable to depend on the library, but sometimes that would unnecessarily pollute the interface. Useful to be aware of, and sometimes worth fixing, but not always.
* A file has not been compiled despite being mentioned in the `.cabal` file. This situation can be because the file is unused, or the `stack` compilation was incomplete. I recommend compiling both benchmarks and tests to avoid this warning where possible - running `weeder . --build` will use a suitable command line.

Beware of conditional compilation (e.g. `CPP` and the [Cabal `flag` mechanism](https://www.haskell.org/cabal/users-guide/developing-packages.html#configurations)), as these may mean that something is currently a weed, but in different configurations it is not.

I recommend fixing the warnings relating to `other-modules` and files not being compiled first, as these may cause other warnings to disappear.

## What else should I use?

Weeder detects dead exports, which can be deleted. To get the most code deleted from removing an export, use:

* GHC with `-fwarn-unused-binds -fwarn-unused-imports`, which finds unused definitions and unused imports in a module.
* [HLint](https://github.com/ndmitchell/hlint#readme), looking for "Redundant extension" hints, which finds unused extensions.

## Ignoring weeds

If you want your package to be detected as "weed free", but it has some weeds you know about but don't consider important, you can add a `.weeder.yaml` file adjacent to the `stack.yaml` with a list of exclusions. To generate an initial list of exclusions run `weeder . --yaml > .weeder.yaml`.

You may wish to generalise/simplify the `.weeder.yaml` by removing anything above or below the interesting part. As an example of the [`.weeder.yaml` file from `ghcid`](https://github.com/ndmitchell/ghcid/blob/master/.weeder.yaml):

```yaml
- message: Module reused between components
- message:
  - name: Weeds exported
  - identifier: withWaiterPoll
```

This configuration declares that I am not interested in the message about modules being reused between components (that's the way `ghcid` works, and I am aware of it). It also says that I am not concerned about `withWaiterPoll` being a weed - it's a simplified method of file change detection I use for debugging, so even though it's dead now, I sometimes do switch to it.

## Running with Continuous Integration

Before running Weeder on your continuous integration (CI) server, you should first ensure there are no existing weeds. One way to achieve that is to ignore existing hints by running `weeder . --yaml > .weeder.yaml` and checking in the resulting `.weeder.yaml`.

On the CI you should then run `weeder .` (or `weeder . --build` to compile as well). To avoid the cost of compilation you may wish to fetch the [latest Weeder binary release](https://github.com/ndmitchell/weeder/releases/latest). For certain CI environments there are helper scripts to do that.

**Travis:** Execute the following command:

    curl -sL https://raw.github.com/ndmitchell/weeder/master/misc/travis.sh | sh -s .

The arguments after `-s` are passed to `weeder`, so modify the final `.` if you want other arguments.

**Appveyor:** Add the following statement to `.appveyor.yml`:

    - ps: Invoke-Command ([Scriptblock]::Create((Invoke-WebRequest 'https://raw.githubusercontent.com/ndmitchell/weeder/master/misc/appveyor.ps1').Content)) -ArgumentList @('.')

The arguments inside `@()` are passed to `weeder`, so add new arguments surrounded by `'`, space separated - e.g. `@('.' '--build')`.

## What about Cabal users?

Weeder requires the textual `.hi` file for each source file in the project. Stack generates that already, so it was easy to integrate in to. There's no reason that information couldn't be extracted by either passing flags to Cabal, or converting the `.hi` files afterwards. I welcome patches to do that integration.

## What about false positives?

Weeder strives to avoid incorrectly warning about something that is required, if you find such an instance please report it on [the issue tracker](https://github.com/ndmitchell/weeder/issues). Unfortunately there are some cases where there are still false positives, as GHC doesn't put enough information in the `.hi` files:

**Data.Coerce** If you use `Data.Coerce.coerce` the constructors for the data type must be in scope, but if they aren't used anywhere other than automatically by `coerce` then Weeder will report unused imports. You can ignore such warnings by adding `- message: Unused import` to your `.weeder.yaml` file.

**Declaration QuasiQuotes** If you use a declaration-level quasi-quote then weeder won't see the use of the quoting function, potentially leading to an unused import warning, and marking the quoting function as a weed. The only solution is to ignore the entries with a `.weeder.yaml` file.

**Stack extra-deps** Packages marked extra-deps in your `stack.yaml` will be weeded, due to a bug in [`stack`](https://github.com/commercialhaskell/stack/issues/3258). The only solution is to ignore the packages with a `.weeder.yaml` file.
