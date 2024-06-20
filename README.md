<img src="http://gfxmonk.net/dist/status/project/opam2nix.png">

# opam2nix

opam2nix, as its name suggests, takes in [opam][] inputs and generates [nix][] expressions. There are many `something2nix` tools for various languages, this is the OCaml one.

### Note on previous versions:

opam2nix version 0.x was active from ~2015-2019. Version 1.0 introduced a simpler but very different usage. In particular the [opam2nix-packages](https://github.com/gfxmonk/opam2nix-packages) repository was only necessary prior to 1.0, so its documentation still describes that version.

## Examples

The easiest way to get started is to check out the [examples/](./examples/) directory. It's got small, working examples that you can probably adapt to your own use very easily.

## Getting it

opam2nix is not yet part of nixpkgs, so the easiest way to use it is to make a simple file which imports it:

```
$ cat opam2nix.nix
import (builtins.fetchTarball "https://github.com/timbertson/opam2nix/archive/v1.tar.gz")
```

This will always build the latest version, cached based on your nix settings. If you prefer to specify an exact version, you can use `fetchFromGitHub`, or use a tool like [nix-wrangle][].

### Using it

`opam2nix` has two interfaces. The first one is the commandline interface, which you'll use to resolve your dependencies into a concrete package set from the requirements in your opam file (`my-package.opam` for example):

```
$ "$(nix-build --no-out-link ./opam2nix.nix)/bin/opam2nix" resolve --ocaml-version 4.14.2 ./my-package.opam
```

This selects a specific version of all transitive dependencies using the latest version of the opam repository by default, although you can specify a specific revision (with `--repo-commit=COMMIT` / `$OPAM_REPO_COMMIT=COMMIT`), or skip updating by passing `--repo-commit=HEAD`.

Secondly, you'll need to use the nix API to actually build the generated package set:

```
$ cat default.nix
with import <nixpkgs> {};
let
  ocaml = ocaml-ng.ocamlPackages_4_14.ocaml;
  opam2nix = import ./opam2nix.nix {};
  selection = opam2nix.build {
    inherit ocaml;
    selection = ./opam-selection.nix;
    src = ./.;
  };
in
selection.my-package
```

That's it! Whenever you change your dependencies you'll need to re-generate `opam-selection.nix`, otherwise you can just use `nix-build`, `nix-shell` etc.

## API

The `opam2nix` nix derivation exposes these functions (as `passthru` attributes):

 - `build`: build a package set, and return the full selections (an attrset with all opam package names as keys, and derivations as values)
 - `buildInputs`: as above, but returns an array of derivations (i.e. the `attrValues`)

Both of these functions accept a single attrset argument, containing:

 - `ocaml`: required, the version must match the version specified when creating `opam-dependencies.nix`
 - `src`: if building a package that doesn't live in the official `opam-repository` (called "direct" packages by opam2nix), you must provide `src`. This can either be:
    - a single source value, in which case it's used by all (typically one) direct packages
    - an attrset with a key for each direct package, for when you're building multiple direct packages with distinct sources
 - `override`: optional, an attrset to supply overrides to selected packages. This can contain as many keys as you like, overrides will only apply to packages that are being built. Override values are described below.

Additionally, there's a third `resolve` function. It takes two arguments:

 - an attributeset which may contain the same arguments passed to `build` / `buildInputs`. Only the `ocaml` and `selection` attributes are required / used, the other attributes are ignored so that you can reuse the same attrset.
 - an array of arguments to be passed to `opam2nix resolve`. The `--dest` and `--ocaml-version` arguments are provided for you, so you don't need to specify those.

This does not produce a derivation, instead it produces a shell environment. If you expose this as e.g. an `update` attribute in your `default.nix`, you can use it like so:

```
nix-shell -A update default.nix
```

This will run `opam2nix resolve` and then exit, and since it's in a shell it is allowed to do impure things like fetching items from the internet, and writing to your local selections file.

This is just a convenience, but in certain situations it becomes quite useful. In particular when you need to pass in the paths to many `opam` files which come from nix derivations (e.g. the result of `fetchFromGitHub`). You can do this by `nix-build`-ing each one individually and passing them in, but using the `resolve` function lets you pass these dependencies in as simple expressions and have nix build everything in a single execution.

### Overrides

The `overrides` value is applied to `pkgs.callPackage`, so it can be either a path or a function. The function can require named arguments from `pkgs`, plus the additional properties `selection` and `opam2nix`. All of these are optional, if your `overrides` object requires no dependencies it can simply be a function accepting no named arguments (i.e. `{}: ...`).

The return value of the `overrides` function must be an attrset, with keys matching opam package names. Excess names (i.e. packages which aren't being built) are allowed, and ignored.

Each value in the returned attrset must be a function accepting a single argument, the base derivation for this opam package. By convention, this is named `super`. When invoked, this function must return a new derivation, typically by invoking `super.overrideAttrs`.

An example `override` argument looks like this:

```
{ pkgs, selection }: {
	nocrypto = super: super.overrideAttrs (attrs: {
		buildPhase = "export OCAMLRUNPARAM=b; " + attrs.buildPhase;
		hardeningDisable = [ "stackprotector" ];
		propagatedBuildInputs = attrs.propagatedBuildInputs ++ [ selection.ocamlfind pkgs.libffi ];
	});
}
```

(nocrypto doesn't actually need ocamlfind and libffi added to its build inputs, this is just for illustrative purposes).

# How do opam depexts work?

For depexts of `os-distribution = "nixos"`, those will become mandatory (and they're resolved as attributes on whatever pkgs is in use).

If a package has depexts but none of them are for nixos, they'll all become optional depexts - if there is a nixpkgs attribute matching that name it'll be used, otherwise no dep is added.

[nix-wrangle]: https://github.com/timbertson/nix-wrangle/
[opam]: https://opam.ocaml.org
[nix]: http://nixos.org/nix/
