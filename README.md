# evcxr-lab

Research projects with [evcxr](https://github.com/evcxr/evcxr) — a Rust REPL and Jupyter kernel.

The whole environment (evcxr, the Rust toolchain, Jupyter and the kernel registration) is provided
by a Nix flake: no toolchain, package or Jupyter kernel is installed into `$HOME`, and nothing
depends on `rustup`. (`:dep` still uses Cargo's normal `~/.cargo` download cache, which is a
feature — dependencies stay warm between sessions.)

## Usage

```sh
nix develop          # or: direnv allow
jupyter lab          # open notebooks/, pick the "Rust" kernel
evcxr                # or just use the REPL
```

Without entering the shell:

```sh
nix run .#lab        # jupyter lab with the Rust kernel registered
nix run nixpkgs#evcxr
```

## What the flake provides

| Output | Contents |
| --- | --- |
| `devShells.default` | evcxr, cargo/rustc/clippy/rustfmt/rust-analyzer, Jupyter, `pkg-config` |
| `packages.kernelspec` | Jupyter data dir containing only the `rust` kernel |
| `packages.jupyter` | Python env with jupyterlab, notebook, nbconvert, nbclient |
| `packages.lab` (`packages.default`) | `jupyter lab` wrapper with the kernel pre-registered |
| `checks.kernelspec` | asserts the kernel's `argv[0]` is the wrapped, runnable binary |

The kernel is registered declaratively through `JUPYTER_PATH`, so **do not run
`evcxr_jupyter --install`** — the kernelspec lives in the read-only Nix store and is already
wired up. Inside `nix develop`, Jupyter's config, state and sockets are additionally redirected
to `.jupyter/` inside the repo; `nix run .#lab` only registers the kernel and otherwise leaves
Jupyter's own defaults alone.

`rustc`/`cargo` in the shell are the same 1.x nixpkgs build that the `evcxr` wrapper uses
internally, so notebooks and `cargo build` agree on the toolchain.

## Notebook notes

- `notebooks/00-getting-started.ipynb` is the smoke test: evaluation, cross-cell persistence,
  `:dep` from crates.io, and custom HTML output.
- `:dep` accepts anything Cargo does, e.g.
  `:dep polars = { version = "0.4", features = ["lazy"] }`.
- Relative `path` dependencies resolve against the **notebook's own directory** (the kernel's
  working directory), so a crate kept in this repo is loaded with
  `:dep mylib = { path = "../crates/mylib" }`.
- An `evcxr.toml` in the kernel's working directory is picked up automatically and can set
  `[dependencies]` plus `[evcxr]` options (`prelude`, `opt_level`, `offline_mode`, `tmpdir`).
- A type with an `evcxr_display(&self)` method controls its own rendering (HTML, SVG, PNG, …).
  Bind the value to a variable and end the cell with that variable. Observed with evcxr 0.22:
  moving a variable that the kernel is persisting straight into the final expression
  (`Thing { field }`) makes it render with `Debug` instead, which fails outright for types that
  don't implement `Debug`.
- Kernel interrupts are not supported by evcxr; restart the kernel instead.
- `:dep` crates that link against system libraries need those libraries in the shell — add them
  to `packages` in `flake.nix` (with `pkg-config` already present) and re-enter `nix develop`.

## Re-running a notebook headlessly

```sh
jupyter nbconvert --to notebook --execute --inplace notebooks/00-getting-started.ipynb
```
