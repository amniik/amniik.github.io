---
title: "Rust Notes: Cargo"
categories:
  - Learning Notes
  - Rust
tags: [rust, cargo, rust-lang, build-system, dependency-management, rust-workspace]
description: "A practical guide to Rust Cargo."

toc: true
---

# Rust Cargo: Cargo.toml, Cargo.lock, Dependencies, Features, Workspaces, and More

Cargo is Rust's **package manager and build system**.

It is responsible for much more than compiling Rust code. Cargo manages dependencies, resolves dependency versions, builds packages, runs tests, handles features, manages workspaces, creates distributable packages, and can publish packages to registries such as crates.io.

A useful mental model is:

```text
                         Cargo
                           |
        +------------------+------------------+
        |                  |                  |
   Build system       Dependency manager   Package manager
        |                  |                  |
   rustc/rustdoc      Cargo.toml           crates.io
        |              Cargo.lock
        |                  |
        +----------+-------+
                   |
              Rust project
```

The two files that are most important to understand are:

```text
Cargo.toml
Cargo.lock
```

They have very different purposes.

---

# 1. Cargo.toml vs Cargo.lock

The easiest way to remember the difference is:

```text
Cargo.toml
    =
"What dependencies and configuration does my project want?"

Cargo.lock
    =
"What exact dependency versions did Cargo resolve?"
```

For example, `Cargo.toml` might say:

```toml
[dependencies]
serde = "1"
tokio = "1"
```

This specifies version requirements.

It does **not** necessarily mean:

```text
serde = exactly version 1.0.x
tokio = exactly version 1.x.x
```

Instead, these are version requirements that Cargo resolves according to its dependency resolver.

The resulting exact versions are recorded in `Cargo.lock`.

---

# 2. What Is a Cargo Package?

A Cargo package normally contains:

```text
my-project/
├── Cargo.toml
└── src/
    └── main.rs
```

For a library:

```text
my-library/
├── Cargo.toml
└── src/
    └── lib.rs
```

The `Cargo.toml` file is called the **manifest** of the package.

Cargo uses this manifest to understand how the package should be built and what it depends on.

---

# 3. Creating a Package

Create a binary package:

```bash
cargo new my-app
```

This creates:

```text
my-app/
├── Cargo.toml
└── src/
    └── main.rs
```

Create a library:

```bash
cargo new my-library --lib
```

This creates:

```text
my-library/
├── Cargo.toml
└── src/
    └── lib.rs
```

Cargo provides commands such as `new`, `build`, `check`, `test`, `run`, `add`, `remove`, `update`, `tree`, `metadata`, `package`, and `publish`.

---

# 4. Basic Cargo.toml

A simple `Cargo.toml` might look like:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = "1"
tokio = "1"
```

There are two major sections here:

```toml
[package]
```

and:

```toml
[dependencies]
```

The first describes the package.

The second describes dependencies.

---

# 5. The `[package]` Section

The `[package]` section describes the package itself.

A common example:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
description = "My Rust application"
license = "MIT"
repository = "https://github.com/example/my-app"
```

Important fields include:

```toml
name
version
edition
rust-version
description
license
repository
homepage
documentation
readme
authors
```

Cargo's manifest reference defines these and many other package fields.

---

# 6. `name`

```toml
[package]
name = "my-app"
```

This is the package name.

It is also used when selecting the package:

```bash
cargo build -p my-app
```

For a library package, the crate name is generally derived from the package name, with hyphens converted to underscores.

For example:

```toml
name = "my-library"
```

can be imported as:

```rust
use my_library::SomeType;
```

---

# 7. `version`

```toml
version = "0.1.0"
```

This is the package version.

Cargo uses semantic versioning conventions for package versions.

For example:

```text
0.1.0
0.2.0
1.0.0
1.2.3
```

When publishing a crate, the version is part of the crate's identity.

---

# 8. `edition`

```toml
edition = "2024"
```

The Rust edition controls language behavior and compatibility rules.

Common editions include:

```text
2015
2018
2021
2024
```

The edition is **not the same thing as the compiler version**.

For example:

```toml
edition = "2024"
```

does not mean:

```text
Rust compiler version 2024
```

It means:

> Compile this package using the Rust 2024 edition rules.

---

# 9. `rust-version`

You can specify the minimum Rust version supported by your package:

```toml
rust-version = "1.85"
```

This tells Cargo:

> This package requires at least Rust 1.85.

This is particularly useful for libraries because users may have different compiler versions.

---

# 10. Dependencies

Dependencies are normally declared in:

```toml
[dependencies]
```

For example:

```toml
[dependencies]
serde = "1"
tokio = "1"
anyhow = "1"
```

This means:

```text
my application
    |
    +-- serde
    +-- tokio
    +-- anyhow
```

Cargo resolves the complete dependency graph.

---

# 11. Dependency Version Requirements

You will commonly see:

```toml
serde = "1"
```

or:

```toml
serde = "1.0"
```

or:

```toml
serde = "1.0.200"
```

These are **version requirements**, not necessarily exact versions.

For example:

```toml
serde = "1"
```

allows compatible versions within the `1.x` line according to Cargo's version requirement rules.

The exact selected version is recorded in `Cargo.lock`.

---

# 12. Exact Dependency Versions

You can request an exact version:

```toml
serde = "=1.0.219"
```

The `=` means:

```text
exactly 1.0.219
```

This is much more restrictive than:

```toml
serde = "1"
```

In most projects, normal compatible version requirements are preferred.

---

# 13. Dependency Sources

Dependencies can come from different sources.

## crates.io

```toml
[dependencies]
serde = "1"
```

## Git repository

```toml
[dependencies]
some-library = {
    git = "https://github.com/example/some-library"
}
```

## Git branch

```toml
[dependencies]
some-library = {
    git = "https://github.com/example/some-library",
    branch = "main"
}
```

## Git tag

```toml
[dependencies]
some-library = {
    git = "https://github.com/example/some-library",
    tag = "v1.2.0"
}
```

## Git revision

```toml
[dependencies]
some-library = {
    git = "https://github.com/example/some-library",
    rev = "abcdef123456"
}
```

## Local path

```toml
[dependencies]
my-library = {
    path = "../my-library"
}
```

---

# 14. Dependency Aliasing

You can give a dependency a different name:

```toml
[dependencies]
my_serde = {
    package = "serde",
    version = "1"
}
```

Then:

```rust
use my_serde::Serialize;
```

The `package` field specifies the actual package name while the key specifies the dependency name used by your crate.

---

# 15. Dependency Features

Dependencies can expose Cargo features.

For example:

```toml
[dependencies]
tokio = {
    version = "1",
    features = ["rt", "macros", "net"]
}
```

This means:

```text
tokio
 |
 +-- rt
 +-- macros
 +-- net
```

Cargo features are compile-time configuration options.

---

# 16. Optional Dependencies

A dependency can be optional:

```toml
[dependencies]
serde = {
    version = "1",
    optional = true
}
```

Then the dependency is not necessarily enabled for every build.

A feature can enable it:

```toml
[features]
json = ["dep:serde"]
```

Now:

```bash
cargo build --features json
```

enables `serde`.

The dependency graph becomes:

```text
json feature
     |
     v
   serde
```

---

# 17. Cargo Features

Features are compile-time switches.

For example:

```toml
[features]
default = ["logging"]
logging = []
async = []
json = []
```

You can enable a feature:

```bash
cargo build --features async
```

Multiple features:

```bash
cargo build --features "async json"
```

All features:

```bash
cargo build --all-features
```

Disable default features:

```bash
cargo build --no-default-features
```

Features are commonly used for:

* optional functionality
* optional dependencies
* platform support
* `no_std` support
* different backends
* different implementations

---

# 18. `default` Features

A special feature named `default` can specify features enabled automatically.

```toml
[features]
default = ["std"]
std = []
no_std = []
```

Running:

```bash
cargo build
```

enables:

```text
default
  |
  +-- std
```

To disable default features:

```bash
cargo build --no-default-features
```

You can then explicitly enable another configuration:

```bash
cargo build --no-default-features --features no_std
```

---

# 19. Conditional Compilation

Features become especially useful with Rust's `cfg` attributes.

Cargo:

```toml
[features]
logging = []
```

Rust:

```rust
#[cfg(feature = "logging")]
fn log_message() {
    println!("logging enabled");
}
```

When the feature is enabled:

```bash
cargo build --features logging
```

the function is compiled.

Without the feature, it is not compiled.

This is compile-time configuration:

```text
Cargo feature
      |
      v
rustc --cfg
      |
      v
#[cfg(...)]
      |
      v
selected code
```

---

# 20. Development Dependencies

Dependencies used only for development go under:

```toml
[dev-dependencies]
```

For example:

```toml
[dev-dependencies]
pretty_assertions = "1"
criterion = "0.5"
```

These dependencies are typically used by:

* tests
* benchmarks
* examples

They are not normal runtime dependencies of the package.

---

# 21. Build Dependencies

Dependencies required by a build script go under:

```toml
[build-dependencies]
```

For example:

```toml
[build-dependencies]
cc = "1"
bindgen = "0.71"
```

If you have:

```text
build.rs
```

and it needs `cc`, then:

```toml
[build-dependencies]
cc = "1"
```

is appropriate.

This is especially common when building native code or generating Rust bindings.

---

# 22. Target-Specific Dependencies

Dependencies can be conditional on the target platform.

For example:

```toml
[target.'cfg(windows)'.dependencies]
windows = "0.60"
```

For Linux:

```toml
[target.'cfg(target_os = "linux")'.dependencies]
libc = "0.2"
```

You can also target architectures:

```toml
[target.'cfg(target_arch = "x86_64")'.dependencies]
some-x86-library = "1"
```

This is very useful for systems programming.

---

# 23. Environment-Specific Configuration

Cargo also supports configuration outside `Cargo.toml`.

For example:

```text
.cargo/
└── config.toml
```

A project might contain:

```toml
[build]
jobs = 4
```

Cargo configuration is different from package manifest configuration.

A useful distinction is:

```text
Cargo.toml
    |
    +-- describes the package

.cargo/config.toml
    |
    +-- configures Cargo's behavior
```

Cargo searches for configuration files hierarchically from the current directory and its parents, up to the user's Cargo home directory.

---

# 24. Cargo Aliases

You can define command aliases in `.cargo/config.toml`:

```toml
[alias]
c = "check"
b = "build"
t = "test"
```

Then:

```bash
cargo c
```

means:

```bash
cargo check
```

and:

```bash
cargo t
```

means:

```bash
cargo test
```

This can be useful for large projects with frequently used command combinations.

---

# 25. Cargo Profiles

Cargo profiles control compiler settings.

The most common profiles are:

```text
dev
release
test
bench
```

You can configure release builds:

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

Then:

```bash
cargo build --release
```

uses these settings.

---

# 26. Debug vs Release

Normal:

```bash
cargo build
```

uses the development profile.

Release:

```bash
cargo build --release
```

uses the release profile.

Conceptually:

```text
cargo build
     |
     v
dev profile
     |
     +-- faster compilation
     +-- debugging information
     +-- less optimization


cargo build --release
     |
     v
release profile
     |
     +-- more optimization
     +-- slower compilation
     +-- optimized binary
```

---

# 27. Important Release Options

A common release configuration is:

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true
```

### `opt-level`

Controls optimization.

Common values include:

```text
0
1
2
3
"s"
"z"
```

For example:

```toml
opt-level = 3
```

means aggressive optimization.

---

# 28. Link-Time Optimization

You can enable LTO:

```toml
[profile.release]
lto = true
```

LTO stands for:

> Link-Time Optimization

It allows optimization across crate boundaries during linking.

This can improve runtime performance or binary size, at the cost of longer build times.

---

# 29. `codegen-units`

You can configure:

```toml
[profile.release]
codegen-units = 1
```

Fewer codegen units can allow better optimization but can increase compilation time.

A typical performance-oriented configuration might therefore be:

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

---

# 30. `Cargo.lock`

Now we reach one of the most important Cargo concepts.

`Cargo.toml` specifies **requirements**.

`Cargo.lock` records the **resolved dependency graph**.

For example:

```toml
[dependencies]
tokio = "1"
serde = "1"
```

Cargo might resolve:

```text
tokio 1.48.0
serde 1.0.228
```

and record the exact versions and source information in:

```text
Cargo.lock
```

---

# 31. Why Cargo.lock Exists

Imagine your project says:

```toml
serde = "1"
```

Today:

```text
serde 1.0.220
```

is selected.

Six months later:

```text
serde 1.0.230
```

is available.

Without a lockfile, different dependency resolution could potentially result in different versions.

With:

```text
Cargo.lock
```

Cargo can reproduce the resolved dependency graph.

Think:

```text
Cargo.toml
    |
    | version requirements
    v
Dependency resolver
    |
    v
Cargo.lock
    |
    | exact resolved graph
    v
Build
```

---

# 32. Should Cargo.lock Be Committed?

For applications and binaries, generally **yes**.

For libraries, Cargo has historically treated lockfiles differently because downstream users resolve library dependencies as part of their own dependency graph.

A useful rule is:

```text
Application / binary
    -> commit Cargo.lock

Published library
    -> usually don't commit Cargo.lock as part of the library repository
       unless there is a project-specific reason
```

The exact behavior and packaging rules can differ depending on the package and publishing workflow, so the important point is to understand the distinction between an application's reproducible dependency set and a library's role in another project's dependency graph.

---

# 33. Updating Dependencies

You can run:

```bash
cargo update
```

This asks Cargo to update dependencies according to the requirements in `Cargo.toml`.

For example:

```toml
serde = "1"
```

may allow Cargo to move from one compatible `1.x` release to another.

The lockfile changes accordingly.

---

# 34. Updating One Dependency

You can update a particular package:

```bash
cargo update -p serde
```

This is useful when you don't want to update the entire dependency graph.

---

# 35. `cargo tree`

One of the most useful Cargo commands for understanding dependencies is:

```bash
cargo tree
```

For example:

```text
my-app
├── serde
│   └── serde_derive
└── tokio
    └── ...
```

This shows the dependency graph.

When debugging dependency problems, `cargo tree` is extremely useful.

---

# 36. Duplicate Dependencies

Suppose:

```text
my-app
├── library-a
│   └── foo 1.0
└── library-b
    └── foo 2.0
```

Cargo may need to compile two versions:

```text
foo 1.x
foo 2.x
```

You can investigate this with:

```bash
cargo tree
```

or:

```bash
cargo tree -d
```

The `-d` option is useful for identifying duplicate package versions.

---

# 37. `cargo metadata`

For tools and scripts, another important command is:

```bash
cargo metadata
```

For machine-readable output:

```bash
cargo metadata --format-version 1
```

It provides information about:

* packages
* dependencies
* workspace members
* workspace root
* target directory
* package IDs

This is useful when building tooling around Cargo projects.

---

# 38. `cargo check`

During development, prefer:

```bash
cargo check
```

when you only want to know whether the code compiles.

It usually avoids generating the final executable.

For a workspace:

```bash
cargo check --workspace
```

A common development loop is:

```bash
cargo check
cargo test
cargo clippy
```

---

# 39. `cargo build`

Build the project:

```bash
cargo build
```

Release build:

```bash
cargo build --release
```

Build a particular package:

```bash
cargo build -p my-crate
```

Build an entire workspace:

```bash
cargo build --workspace
```

---

# 40. `cargo run`

For binary crates:

```bash
cargo run
```

Pass arguments to the application after `--`:

```bash
cargo run -- --config config.toml
```

For a workspace:

```bash
cargo run -p my-cli
```

---

# 41. `cargo test`

Run tests:

```bash
cargo test
```

Run workspace tests:

```bash
cargo test --workspace
```

Run a particular test:

```bash
cargo test test_name
```

Run tests for one package:

```bash
cargo test -p my-library
```

---

# 42. `cargo clippy`

Clippy is Rust's linting tool.

Run:

```bash
cargo clippy
```

For a workspace:

```bash
cargo clippy --workspace
```

In CI, you might use:

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

This treats warnings as errors.

---

# 43. `cargo fmt`

Formatting is handled by rustfmt:

```bash
cargo fmt
```

Check formatting without modifying files:

```bash
cargo fmt -- --check
```

A common CI pipeline is:

```bash
cargo fmt -- --check
cargo check --workspace
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
```

---

# 44. `cargo add`

Instead of manually editing:

```toml
[dependencies]
```

you can use:

```bash
cargo add serde
```

You can specify a version:

```bash
cargo add serde@1
```

Features:

```bash
cargo add tokio -F rt -F macros
```

Development dependency:

```bash
cargo add --dev criterion
```

Optional dependency:

```bash
cargo add serde --optional
```

Git dependency:

```bash
cargo add some-library --git https://github.com/example/some-library
```

Path dependency:

```bash
cargo add my-library --path ../my-library
```

Cargo's `cargo add` command supports these dependency source and feature options.

---

# 45. `cargo remove`

Remove a dependency:

```bash
cargo remove serde
```

Development dependency:

```bash
cargo remove --dev criterion
```

---

# 46. `cargo clean`

Remove build artifacts:

```bash
cargo clean
```

This removes the generated build output, typically under:

```text
target/
```

It does not mean:

```text
delete Cargo.lock
```

Cargo's dependency resolution information is separate from build artifacts.

---

# 47. `target/`

Cargo normally puts build artifacts into:

```text
target/
```

For example:

```text
target/
├── debug/
│   ├── my-app
│   ├── deps/
│   └── incremental/
│
└── release/
    ├── my-app
    └── deps/
```

You generally do not commit `target/` to Git.

A typical `.gitignore` contains:

```gitignore
/target
```

---

# 48. Cargo Workspace

A workspace is a collection of packages managed together.

For example:

```text
project/
├── Cargo.toml
├── Cargo.lock
│
├── core/
│   ├── Cargo.toml
│   └── src/
│
├── network/
│   ├── Cargo.toml
│   └── src/
│
└── cli/
    ├── Cargo.toml
    └── src/
```

Root `Cargo.toml`:

```toml
[workspace]
resolver = "3"

members = [
    "core",
    "network",
    "cli",
]
```

Cargo workspaces provide:

* shared `Cargo.lock`
* shared `target`
* workspace-wide commands
* shared dependency declarations
* shared package metadata
* workspace-level profiles

---

# 49. Workspace Dependencies

A workspace can centralize dependency versions:

```toml
[workspace.dependencies]
serde = "1"
tokio = "1"
anyhow = "1"
thiserror = "2"
```

A member can inherit them:

```toml
[dependencies]
serde = { workspace = true }
tokio = { workspace = true }
```

This avoids repeating versions across many crates.

Important:

```toml
[workspace.dependencies]
serde = "1"
```

does **not** automatically add `serde` to every crate.

The member still needs:

```toml
serde = { workspace = true }
```

---

# 50. Workspace Package Metadata

The workspace can also centralize package metadata:

```toml
[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"
repository = "https://github.com/example/project"
```

A member can inherit:

```toml
[package]
name = "core"

version.workspace = true
edition.workspace = true
license.workspace = true
repository.workspace = true
```

This becomes very useful in large repositories.

---

# 51. Virtual Workspace

A workspace does not necessarily need a root package.

For example:

```toml
[workspace]
resolver = "3"

members = [
    "crates/core",
    "crates/network",
    "crates/cli",
]
```

This is called a **virtual workspace**.

The root is simply the workspace.

---

# 52. Cargo Workspace Commands

Build everything:

```bash
cargo build --workspace
```

Check everything:

```bash
cargo check --workspace
```

Test everything:

```bash
cargo test --workspace
```

Run Clippy:

```bash
cargo clippy --workspace
```

Build one package:

```bash
cargo build -p core
```

Run one binary:

```bash
cargo run -p cli
```

---

# 53. Build Scripts

Cargo supports build scripts through:

```text
build.rs
```

Example:

```text
my-project/
├── Cargo.toml
├── build.rs
└── src/
    └── main.rs
```

Cargo runs the build script before compiling the package.

A build script can be useful for:

* compiling C/C++ code
* generating Rust source
* finding system libraries
* generating bindings
* communicating build-time information to `rustc`

For example:

```toml
[build-dependencies]
cc = "1"
```

and:

```rust
fn main() {
    cc::Build::new()
        .file("src/native.c")
        .compile("native");
}
```

This is particularly relevant in systems programming.

---

# 54. Cargo and Native Libraries

When working with Rust systems software, you may encounter:

```text
build.rs
cc
bindgen
pkg-config
system libraries
linker configuration
```

Cargo can coordinate these build steps.

For example:

```text
Cargo
 |
 +-- build.rs
 |      |
 |      +-- find libfoo
 |      +-- generate bindings
 |      +-- compile C code
 |
 +-- rustc
 |
 +-- linker
```

This is one reason Cargo is more than a simple package downloader.

---

# 55. Cargo Registries

The default public Rust package registry is:

```text
crates.io
```

You can declare:

```toml
[dependencies]
serde = "1"
```

and Cargo normally obtains the package from the configured registry.

Cargo also supports alternative registries, which are useful for organizations with private packages.

This is especially relevant for companies with internal Rust infrastructure.

---

# 56. Publishing a Crate

Before publishing, you can inspect the package:

```bash
cargo package
```

You can see what files would be included:

```bash
cargo package --list
```

Then publish:

```bash
cargo publish
```

Publishing creates a distributable `.crate` archive and uploads it to the configured registry, normally crates.io.

---

# 57. `include` and `exclude`

Cargo lets you control which files are included in a published package.

For example:

```toml
[package]
name = "my-library"
version = "0.1.0"
include = [
    "src/**",
    "Cargo.toml",
    "README.md",
]
```

You can also use:

```toml
exclude = [
    "tests/data/**",
]
```

This matters when publishing libraries.

---

# 58. Cargo Configuration vs Cargo.toml

This distinction is easy to miss.

### `Cargo.toml`

Describes the package:

```text
package
dependencies
features
targets
workspace
profiles
```

### `.cargo/config.toml`

Configures Cargo itself:

```text
build settings
target
linker
aliases
registries
network settings
rustflags
```

Think:

```text
Cargo.toml
    =
"What is this project?"

.cargo/config.toml
    =
"How should Cargo operate?"
```

---

# 59. Cross Compilation

Cargo can build for another target:

```bash
cargo build --target x86_64-unknown-linux-gnu
```

or:

```bash
cargo build --target aarch64-unknown-linux-gnu
```

The target can also be configured:

```toml
[build]
target = "aarch64-unknown-linux-gnu"
```

in `.cargo/config.toml`.

For systems programming, understanding Cargo's target configuration becomes very important.

---

# 60. Cargo and `rustc`

Cargo is not the Rust compiler.

The compiler is:

```text
rustc
```

Cargo orchestrates the compilation process.

Conceptually:

```text
Cargo
 |
 +-- reads Cargo.toml
 |
 +-- resolves dependencies
 |
 +-- reads Cargo.lock
 |
 +-- prepares build
 |
 +-- invokes rustc
 |
 +-- invokes rustdoc when needed
 |
 +-- invokes linker
 |
 +-- produces artifacts
```

This distinction is important.

You can invoke:

```bash
rustc main.rs
```

without Cargo.

But Cargo gives you the complete package/build/dependency ecosystem.

---

# 61. Cargo as a Build Graph Manager

A useful way to think about Cargo is as a **build graph manager**.

Suppose:

```text
my-app
 |
 +-- library-a
 |      |
 |      +-- serde
 |
 +-- library-b
        |
        +-- tokio
```

Cargo determines:

```text
what must be compiled
what depends on what
which versions are required
which features are enabled
which build scripts must run
which artifacts can be reused
```

Then it executes the required build steps.

---

# 62. Dependency Resolution

Suppose:

```text
my-app
 |
 +-- library-a
 |      |
 |      +-- foo ^1.0
 |
 +-- library-b
        |
        +-- foo ^1.2
```

Cargo tries to find a compatible dependency graph.

It might resolve:

```text
foo 1.5
```

for both.

But if constraints are incompatible:

```text
library-a -> foo 1.x
library-b -> foo 2.x
```

Cargo may need:

```text
foo 1.x
foo 2.x
```

in the same dependency graph.

This is why `cargo tree` is so useful.

---

# 63. Reproducible Builds

For applications, a committed:

```text
Cargo.lock
```

combined with:

```bash
cargo build --locked
```

can be useful in CI.

`--locked` tells Cargo that it must not change the lockfile.

If the lockfile is missing or dependency resolution would change it, Cargo fails.

This is useful when you want CI to build using exactly the dependency resolution committed to the repository.

---

# 64. `--offline`

You can tell Cargo not to access the network:

```bash
cargo build --offline
```

Cargo then tries to use locally available dependency information and cached packages.

This can be useful for:

* offline development
* restricted build environments
* reproducibility testing
* isolated CI/build systems

But it can fail if the required dependencies are not already available locally.

---

# 65. `--frozen`

You can use:

```bash
cargo build --frozen
```

This is effectively:

```text
--locked
+
--offline
```

So Cargo must:

```text
not modify Cargo.lock
+
not access the network
```

This is useful in controlled build environments.

---

# 66. Important CI Commands

A Rust project's CI might contain:

```bash
cargo fmt -- --check
cargo check --workspace
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
cargo build --workspace --release
```

For stricter reproducibility:

```bash
cargo test --workspace --locked
```

or:

```bash
cargo build --workspace --locked
```

---

# 67. Useful Cargo Command Categories

Instead of memorizing every command, group them mentally.

### Development

```bash
cargo check
cargo build
cargo run
cargo test
cargo fmt
cargo clippy
```

### Dependencies

```bash
cargo add
cargo remove
cargo update
cargo tree
cargo metadata
```

### Workspace

```bash
cargo build --workspace
cargo test --workspace
cargo check --workspace
cargo build -p package
```

### Packaging

```bash
cargo package
cargo publish
```

### Build management

```bash
cargo clean
cargo build --release
cargo build --target ...
```

---

# 68. A Complete Cargo.toml Example

Here is an example combining many of the concepts:

```toml
[package]
name = "my-vmm"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
license = "Apache-2.0"
description = "A small experimental VMM"

[dependencies]
anyhow = "1"
thiserror = "2"

tokio = {
    version = "1",
    optional = true,
    features = ["rt", "macros"]
}

serde = {
    version = "1",
    optional = true,
    features = ["derive"]
}

[dev-dependencies]
pretty_assertions = "1"

[build-dependencies]
cc = "1"

[features]
default = ["std"]

std = []

async = [
    "dep:tokio",
]

json = [
    "dep:serde",
]

[target.'cfg(unix)'.dependencies]
libc = "0.2"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

This single manifest demonstrates:

```text
[package]
[dependencies]
[dev-dependencies]
[build-dependencies]
[features]
[target.*.dependencies]
[profile.release]
```

---

# 69. Complete Project Structure

A larger project might look like:

```text
my-vmm/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE
├── .gitignore
│
├── .cargo/
│   └── config.toml
│
├── crates/
│   ├── vmm-core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── vmm-memory/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── vmm-device/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   └── vmm-net/
│       ├── Cargo.toml
│       └── src/
│
├── tools/
│   └── cli/
│       ├── Cargo.toml
│       └── src/
│
└── target/
```

The important files have different responsibilities:

```text
Cargo.toml
    ↓
project/package/workspace definition

Cargo.lock
    ↓
resolved dependency graph

.cargo/config.toml
    ↓
Cargo configuration

build.rs
    ↓
build-time code

target/
    ↓
generated build artifacts
```

---

# 70. The Most Important Mental Model

When working with Cargo, think in layers:

```text
                         Cargo
                           |
            +--------------+--------------+
            |              |              |
         Manifest       Lockfile       Config
       Cargo.toml      Cargo.lock    .cargo/config.toml
            |              |              |
            v              v              v
      desired setup    exact graph    Cargo behavior
            |
            v
       Dependency
         graph
            |
            v
       Feature graph
            |
            v
        Build graph
            |
            v
          rustc
            |
            v
       linker / artifacts
            |
            v
         target/
```

This mental model makes Cargo much easier to understand.

---

# 71. Cargo.toml vs Cargo.lock vs .cargo/config.toml

A concise comparison:

| File                 | Purpose                                                               |
| -------------------- | --------------------------------------------------------------------- |
| `Cargo.toml`         | Defines the package and desired dependency/configuration requirements |
| `Cargo.lock`         | Records the resolved dependency versions and graph                    |
| `.cargo/config.toml` | Configures Cargo's behavior                                           |
| `build.rs`           | Performs package-specific build-time actions                          |
| `target/`            | Contains generated build artifacts                                    |

The distinction between these files is one of the most important things to understand about Cargo.

---

# 72. What You Should Usually Commit

A typical Rust application repository contains:

```text
Cargo.toml
Cargo.lock
src/
README.md
LICENSE
.cargo/config.toml   # if project-specific
```

and normally does **not** commit:

```text
target/
```

A `.gitignore` commonly contains:

```gitignore
/target
```

---

# 73. What Cargo Actually Does When You Run `cargo build`

Suppose you run:

```bash
cargo build
```

Conceptually, Cargo performs something like:

```text
1. Find Cargo.toml
        |
        v
2. Read package configuration
        |
        v
3. Read dependency requirements
        |
        v
4. Resolve dependency graph
        |
        v
5. Read/update Cargo.lock
        |
        v
6. Determine enabled features
        |
        v
7. Determine build targets
        |
        v
8. Run build scripts
        |
        v
9. Compile dependencies
        |
        v
10. Compile your crate
        |
        v
11. Link binaries/libraries
        |
        v
12. Store artifacts in target/
```

This is why Cargo is much more than:

```text
"download some libraries"
```

It is coordinating the entire Rust build process.

---

# 74. Practical Cargo Workflow

For everyday development, a useful workflow is:

```bash
# Create project
cargo new my-project

# Enter project
cd my-project

# Check code quickly
cargo check

# Build
cargo build

# Run
cargo run

# Test
cargo test

# Format
cargo fmt

# Lint
cargo clippy

# Inspect dependencies
cargo tree

# Add dependency
cargo add serde

# Update dependencies
cargo update

# Release build
cargo build --release
```

For CI:

```bash
cargo fmt -- --check
cargo check --workspace --locked
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --locked
cargo build --workspace --release --locked
```

---

# 75. Key Takeaways

Cargo is Rust's package manager and build system.

The most important files are:

```text
Cargo.toml
Cargo.lock
.cargo/config.toml
```

Remember their roles:

```text
Cargo.toml
    "What does my project require?"

Cargo.lock
    "What exact dependency graph was resolved?"

.cargo/config.toml
    "How should Cargo behave?"
```

The most important sections of `Cargo.toml` are:

```toml
[package]
[dependencies]
[dev-dependencies]
[build-dependencies]
[features]
[target.'cfg(...)'.dependencies]
[workspace]
[workspace.dependencies]
[workspace.package]
[profile.dev]
[profile.release]
[patch.crates-io]
```

The most useful commands to know are:

```bash
cargo new
cargo init
cargo check
cargo build
cargo run
cargo test
cargo fmt
cargo clippy
cargo add
cargo remove
cargo update
cargo tree
cargo metadata
cargo clean
cargo package
cargo publish
```

And for workspaces:

```bash
cargo check --workspace
cargo build --workspace
cargo test --workspace
cargo build -p <package>
cargo test -p <package>
```

The most important concepts to connect together are:

```text
Cargo.toml
     |
     +-- dependencies
     +-- features
     +-- targets
     +-- workspace
     +-- profiles
            |
            v
     dependency resolver
            |
            v
       Cargo.lock
            |
            v
        build graph
            |
            v
          rustc
            |
            v
         target/
```

Once you understand this flow, Cargo becomes much easier to reason about, especially when working on large Rust systems projects with multiple crates, optional backends, platform-specific code, and workspace-wide CI.
