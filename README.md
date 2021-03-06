The haskell-indexer package provides libs for preprocessing Haskell source code
into a representation for easy entity cross-referencing, as well as a frontend
for emitting entities for the [Kythe](https://kythe.io) indexing schema.

This is not an official Google product.

# Supported systems

Indexing hosts:
 - Linux: supported - follow below documentation.
 - Windows, MacOS: didn't try - backend part likely compiles, wiring and Kythe
   frontend likely not (see #38).

Compilers:
  - GHC 8.6.5
  - GHC 8.8.1 (planned)

Stackage:
  - A recent LTS release corresponding to above compilers is supported.
    See `stack-ghcXXX.yml` files.

Previous compilers were supported at some point. Checkout an old repository
state if interested:
  - [GHC 7.10.3](https://github.com/google/haskell-indexer/tree/last-ghc-7.10.3)
  - [GHC 8.4](https://github.com/google/haskell-indexer/tree/last-ghc-8.4)

[![Build Status](https://travis-ci.org/google/haskell-indexer.svg?branch=master)](https://travis-ci.org/google/haskell-indexer)

# Installation

## Stack

Download Stack from http://docs.haskellstack.org

## Kythe

If you want to use the Kythe frontend, you'll need to install it either from
source or from the official release.  The latter is easier, but the web UI has
been removed in recent versions.

### Official Release

Download a [Kythe release](https://github.com/google/kythe/releases) and unpack
it.

```
tar xzf kythe-v0.0.26.tar.gz -C /opt/
rm -r /opt/kythe
ln -s /opt/kythe-v0.0.26 /opt/kythe
chmod -R 755 /opt/kythe/web/ui  # It misses permission by default.
```

Version `v0.0.30` is the latest version that includes the web UI.  If you want a
newer Kythe than this, you'll need to build from source.

If you want to install Kythe in a different location to `/opt/kythe` then you
should also set `KYTHE_DIR` to the location of the installation.

### Building From Source

Clone Kythe from [its GitHub repo](https://github.com/google/kythe/releases) and
follow the [Getting Started guide](https://kythe.io/getting-started/) to build
and install it into `/opt/kythe`.  Then, from within the Kythe clone, build the
web frontend and copy its files into their rightful place:

```
bazel build //kythe/web/ui
mkdir -p /opt/kythe/web/ui
cp -r bazel-bin/kythe/web/ui/resources/public/* /opt/kythe/web/ui
cp -r kythe/web/ui/resources/public/* /opt/kythe/web/ui
chmod -R 755 /opt/kythe/web/ui
```

## Protoc 3

Download the latest [Proto compiler 3
release](https://github.com/google/protobuf/releases), unpack it and place the
binary in the PATH.

```
unzip -j protoc-*-linux-x86_64.zip bin/protoc -d /usr/local/bin/
```

> If you use have Nix installed and you use `stack --nix`, you do not need to do
> this.

## Haskell Indexer Plugin (ghc >= 8.6 only)

Haskell modules can be indexed with a GHC source plugin while building a
project. Whatever build system is in use, indexing can be achieved by
ensuring that the invocations to `ghc` include the flags that enable the
plugin.

For instruction on how to install and use the plugin with `stack`, see
[stack-example/README.md](stack-example/README.md).

If you are using some other build system, the following GHC options are
relevant after the plugin is installed.

* `-package-db <db_path>`: Tells the package database where the plugin has
  been installed. It may be used more than once if the plugin dependencies
  spread through more than one package database.
* `-plugin-package haskell-indexer-plugin`: Tells ghc to expose the package
  containing the plugin, so it can be found when needed.
* `-fplugin Haskell.Indexer.Plugin`: Tells to use the plugin when compiling
  modules.
* `-fplugin-opt Haskell.Indexer.Plugin:-o` and
  `-fplugin-opt Haskell.Indexer.Plugin:<output_path>`: Tell the plugin where
  to place the output of indexing.

# Build the project

Use the following to build and run tests:

```
git clone --recursive https://github.com/google/haskell-indexer.git
cd haskell-indexer
export STACK_YAML=$(readlink -f stack-ghc865.yaml)
stack build && stack test
# To test Kythe frontend:
pushd kythe-verification; stack install && ./test.sh; popd
```

To test all supported stack configurations, do `./run-ghc-tests.sh`.

# Demo

To index a few packages, run:

```bash
export INDEXER_OUTPUT_DIR=/tmp/indexer-output
./build-stack.sh mtlparse cpu
```

The script adds a wrapper for the GHC compiler used by Stack (`stack path --compiler-exe`), does the indexing when `ghc --make` is specified on the command line to build a package. You can run `build-stack.sh` multiple times.

To serve the index at `http://localhost:8080`:

```bash
./serve.sh localhost:8080
```

If you get empty index, look at `$INDEXER_OUTPUT_DIR/*.stderr` files about
possible indexing errors. Also, make sure that the `*.entries` files are not
empty. If they are, it indicates that `ghc_kythe_wrapper` failed to index.

## Indexing using Docker

If you plan to use the Dockerized build feature of stack, please install
Docker. It is also advised to set up a docker wrapper script by following the
instructions at the [stack Docker
security](https://docs.haskellstack.org/en/stable/docker_integration/#security)
section.

The docker image has all C library dependencies so it's possible to use it to
index the whole Stackage snapshot. See `stack-build-docker.sh` for a
comprehensive example of indexing a Stackage snapshot, and serving a Kythe
index.
