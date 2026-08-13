---
title: "Rust Notes: Workspaces"
categories:
  - Learning Notes
  - Rust
tags: [rust, cargo, workspace, crates, dependency-management]
description: "A practical guide to Rust Cargo workspaces."

toc: true
---

# Rust Cargo Workspaces

A **Cargo workspace** is a collection of Rust packages (crates) that Cargo manages together.

Workspaces are especially useful for larger Rust projects where the code is split into multiple crates.

For example:

```text
my-project/
├── Cargo.toml
├── Cargo.lock
├── crates/
│   ├── core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   │
│   ├── network/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   │
│   └── cli/
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
└── target/
```

Each directory can contain an independent Cargo package, while the workspace allows Cargo to manage all of them together.

Cargo workspaces provide a shared `Cargo.lock`, shared `target` directory, workspace-wide commands, and mechanisms for sharing package metadata and dependency declarations.

---

# 1. Why Use a Workspace?

Suppose we have a large project:

```text
my-vmm/
├── vmm-core/
├── vmm-memory/
├── vmm-device/
├── vmm-net/
└── vmm-cli/
```

Each component could be its own crate.

Without a workspace, each crate would be managed independently.

With a workspace:

```text
my-vmm/
├── Cargo.toml
├── Cargo.lock
├── target/
├── vmm-core/
├── vmm-memory/
├── vmm-device/
├── vmm-net/
└── vmm-cli/
```

Cargo can manage all of them as one project.

For example:

```bash
cargo check --workspace
cargo test --workspace
cargo build --workspace
```

This is one of the main reasons workspaces are useful for projects such as libraries, applications split into components, and large systems projects.

---

# 2. Workspace Does Not Mean One Crate

This distinction is important:

```text
Workspace
    |
    +-- Package / crate A
    |
    +-- Package / crate B
    |
    +-- Package / crate C
```

A workspace is **not itself necessarily a crate**.

It is a Cargo-level grouping of packages.

Each member still has its own:

```text
Cargo.toml
src/
```

and its own package identity.

For example:

```text
workspace/
├── Cargo.toml
│
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
│
└── cli/
    ├── Cargo.toml
    └── src/main.rs
```

`core` and `cli` are still separate packages.

---

# 3. The Workspace Root

The workspace normally has a root `Cargo.toml`.

A simple virtual workspace can look like:

```toml
[workspace]
resolver = "3"

members = [
    "core",
    "cli",
]
```

This root `Cargo.toml` only describes the workspace.

It does not need a `[package]` section.

This is called a **virtual workspace**.

The current Rust Book uses `resolver = "3"` in its workspace examples.

---

# 4. Virtual Workspace

A virtual workspace has no package of its own.

Example:

```toml
[workspace]
resolver = "3"

members = [
    "crates/core",
    "crates/network",
    "crates/cli",
]
```

Directory structure:

```text
project/
├── Cargo.toml
│
└── crates/
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

The root is only the workspace.

The actual packages are:

```text
core
network
cli
```

---

# 5. Workspace with a Root Package

A workspace root can also be a package.

For example:

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"

[workspace]
members = [
    "core",
    "cli",
]
```

Now the root has two roles:

```text
workspace root
       +
package
```

This is different from a virtual workspace.

---

# 6. `members`

The `members` field specifies which packages belong to the workspace.

```toml
[workspace]
members = [
    "core",
    "network",
    "cli",
]
```

You can also use paths:

```toml
[workspace]
members = [
    "crates/core",
    "crates/network",
    "tools/cli",
]
```

And globs:

```toml
[workspace]
members = [
    "crates/*",
]
```

Cargo supports glob patterns such as `*` and `?` for workspace members.

---

# 7. `exclude`

Sometimes a directory matches a workspace pattern but should not be part of the workspace.

For example:

```toml
[workspace]
members = [
    "crates/*",
]

exclude = [
    "crates/experimental",
]
```

So:

```text
crates/
├── core/
├── network/
├── storage/
└── experimental/
```

becomes:

```text
workspace members:
    core
    network
    storage

excluded:
    experimental
```

---

# 8. Important: Path Dependencies and Workspace Membership

Suppose:

```toml
[dependencies]
core = { path = "../core" }
```

and `core` is located inside the workspace directory.

Cargo can automatically treat path dependencies inside the workspace as workspace members unless excluded.

This is useful to remember when debugging workspace membership.

---

# 9. Package vs Crate

These terms are often confused.

A **package** is a Cargo concept.

A package contains a `Cargo.toml` and can contain one or more crates.

For example:

```text
package
├── Cargo.toml
└── src/
    ├── lib.rs
    └── main.rs
```

The package may contain:

```text
library crate
binary crate
```

A workspace contains packages:

```text
workspace
├── package A
├── package B
└── package C
```

For everyday Cargo usage, you will often hear "crate" and "package" used loosely, but they are technically different concepts.

---

# 10. Workspace Dependency Management

One of the most useful workspace features is centralizing dependency versions.

Without workspace dependency inheritance, you might have:

```toml
# core/Cargo.toml

[dependencies]
serde = "1.0"
tokio = "1"
```

and:

```toml
# network/Cargo.toml

[dependencies]
serde = "1.0"
tokio = "1"
```

and:

```toml
# cli/Cargo.toml

[dependencies]
serde = "1.0"
tokio = "1"
```

This repeats information.

A workspace can centralize it.

---

# 11. `[workspace.dependencies]`

At the workspace root:

```toml
[workspace]
members = [
    "core",
    "network",
    "cli",
]

[workspace.dependencies]
serde = "1"
tokio = "1"
anyhow = "1"
```

Then a member can inherit the dependency:

```toml
[dependencies]
serde = { workspace = true }
tokio = { workspace = true }
```

Another crate can do:

```toml
[dependencies]
serde = { workspace = true }
anyhow = { workspace = true }
```

The dependency version is therefore defined centrally.

Cargo officially supports workspace dependency inheritance through `[workspace.dependencies]` and `workspace = true`.

---

# 12. Workspace Dependencies Do Not Automatically Become Dependencies

This is a very important point.

If the root contains:

```toml
[workspace.dependencies]
serde = "1"
```

that does **not** mean every crate automatically gets `serde`.

A member still needs:

```toml
[dependencies]
serde = { workspace = true }
```

This is intentional.

Each crate explicitly declares which dependencies it uses.

Think of it as:

```text
workspace.dependencies
        |
        | defines shared version/configuration
        v
member Cargo.toml
        |
        | workspace = true
        v
actual dependency
```

---

# 13. Why Doesn't the Workspace Automatically Add Dependencies?

Because each crate should remain explicit about its own dependencies.

Suppose:

```text
workspace
├── core
├── network
└── cli
```

Only `network` uses:

```text
reqwest
```

It would be undesirable for `core` and `cli` to automatically depend on `reqwest`.

Instead:

```toml
# workspace Cargo.toml

[workspace.dependencies]
reqwest = "0.12"
```

and:

```toml
# network/Cargo.toml

[dependencies]
reqwest = { workspace = true }
```

Only `network` depends on it.

This keeps crate boundaries clear.

---

# 14. Workspace Dependency with Features

The workspace can define common dependency configuration:

```toml
[workspace.dependencies]
tokio = {
    version = "1",
    features = ["rt", "macros"]
}
```

A member can inherit it:

```toml
[dependencies]
tokio = { workspace = true }
```

A member can also add features:

```toml
[dependencies]
tokio = {
    workspace = true,
    features = ["net", "time"]
}
```

Workspace dependency features are additive with features specified by the member.

---

# 15. Workspace Dependencies and Optional Dependencies

Workspace dependencies cannot themselves be declared as `optional`.

For example, this is not allowed at the workspace level:

```toml
[workspace.dependencies]

foo = {
    version = "1",
    optional = true
}
```

Optionality belongs to the member's own dependency declaration.

---

# 16. Workspace Package Metadata

You can also centralize package metadata.

For example:

```toml
[workspace]
members = [
    "core",
    "network",
]

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"
repository = "https://github.com/example/project"
```

Then a member can inherit:

```toml
[package]
name = "core"

version.workspace = true
edition.workspace = true
license.workspace = true
repository.workspace = true
```

Cargo supports workspace inheritance for package metadata such as `version`, `edition`, `license`, `repository`, `description`, `rust-version`, and other package fields.

---

# 17. Why Package Metadata Inheritance Is Useful

Imagine a project with 20 crates.

Without inheritance:

```toml
[package]
version = "0.5.0"
edition = "2024"
license = "Apache-2.0"
repository = "https://github.com/example/project"
```

would be repeated across many crates.

With workspace inheritance:

```toml
[workspace.package]
version = "0.5.0"
edition = "2024"
license = "Apache-2.0"
repository = "https://github.com/example/project"
```

and each crate can use:

```toml
[package]
name = "my-crate"

version.workspace = true
edition.workspace = true
license.workspace = true
repository.workspace = true
```

This makes large repositories easier to maintain.

---

# 18. Workspace Members Can Depend on Each Other

Suppose:

```text
project/
├── Cargo.toml
├── core/
└── cli/
```

and:

```toml
[workspace]
members = [
    "core",
    "cli",
]
```

`cli` can depend on `core`:

```toml
[dependencies]
core = { path = "../core" }
```

The dependency graph becomes:

```text
cli
 |
 v
core
```

Cargo understands that both packages belong to the same workspace.

---

# 19. Workspace Dependency vs Path Dependency

These are different concepts.

A path dependency:

```toml
[dependencies]
core = { path = "../core" }
```

means:

> Find the `core` package at this filesystem path.

Workspace membership means:

> Manage this package as part of the workspace.

You can combine the concepts.

For example:

```toml
[workspace]
members = [
    "core",
    "cli",
]
```

and:

```toml
# cli/Cargo.toml

[dependencies]
core = { path = "../core" }
```

---

# 20. Workspace Commands

One of the biggest practical benefits is running commands across members.

### Check everything

```bash
cargo check --workspace
```

### Build everything

```bash
cargo build --workspace
```

### Test everything

```bash
cargo test --workspace
```

### Run Clippy across the workspace

```bash
cargo clippy --workspace
```

### Build release versions

```bash
cargo build --workspace --release
```

---

# 21. Selecting One Package

You don't always want to build everything.

Use:

```bash
cargo build -p core
```

or:

```bash
cargo test -p core
```

The `-p` option selects a package.

For example:

```bash
cargo test -p network
```

tests only the `network` package.

Cargo supports `-p`/`--package` and `--workspace` for selecting packages.

---

# 22. `cargo run -p`

If the workspace contains a binary crate:

```text
workspace/
├── cli/
├── server/
└── tools/
```

you can run:

```bash
cargo run -p cli
```

or:

```bash
cargo run -p server
```

This is useful when the repository contains several binaries.

---

# 23. `default-members`

A workspace can define which packages should be operated on by default when commands are run from the workspace root.

Example:

```toml
[workspace]
members = [
    "core",
    "network",
    "cli",
]

default-members = [
    "core",
    "cli",
]
```

Now commands from the workspace root can operate on the default members instead of every member.

You can still explicitly use:

```bash
cargo build --workspace
```

to operate on all members.

Cargo documents `default-members` as the package selection used when operating from the workspace root without explicitly selecting packages.

---

# 24. Workspace `Cargo.lock`

A major workspace feature is that packages share a single `Cargo.lock`.

For example:

```text
workspace/
├── Cargo.toml
├── Cargo.lock
├── core/
├── network/
└── cli/
```

There is normally one lockfile at the workspace root.

This gives the project a single resolved dependency graph.

For example:

```text
core
  |
  +-- serde 1.x
  |
  +-- tokio 1.x

network
  |
  +-- serde 1.x
  |
  +-- tokio 1.x

cli
  |
  +-- serde 1.x
```

Cargo resolves these dependencies together.

Cargo workspaces share a common `Cargo.lock` and a common output directory by default.

---

# 25. Workspace `target`

Workspace members normally share the same build output directory:

```text
workspace/
└── target/
```

instead of:

```text
core/target/
network/target/
cli/target/
```

This is useful because Cargo can reuse build artifacts across the workspace.

---

# 26. The Workspace Dependency Graph

A useful way to think about a large workspace is as a graph.

For example:

```text
                 cli
                  |
                  v
               service
              /       \
             v         v
          network     storage
             |          |
             v          v
            core      core
```

Each node is a crate/package.

The workspace gives Cargo one place to manage the complete project.

This is particularly useful for systems projects where different crates represent different layers.

---

# 27. Example: Systems Project

A project similar to a VMM might look like:

```text
vmm/
├── Cargo.toml
│
└── crates/
    ├── vmm-core/
    ├── vmm-memory/
    ├── vmm-device/
    ├── vmm-net/
    └── vmm-cli/
```

Root:

```toml
[workspace]
resolver = "3"

members = [
    "crates/vmm-core",
    "crates/vmm-memory",
    "crates/vmm-device",
    "crates/vmm-net",
    "crates/vmm-cli",
]
```

Shared dependencies:

```toml
[workspace.dependencies]
anyhow = "1"
log = "0.4"
thiserror = "2"
```

Then:

```toml
# crates/vmm-core/Cargo.toml

[package]
name = "vmm-core"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = { workspace = true }
log = { workspace = true }
```

Another crate:

```toml
# crates/vmm-net/Cargo.toml

[package]
name = "vmm-net"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = { workspace = true }
log = { workspace = true }
```

And the CLI:

```toml
# crates/vmm-cli/Cargo.toml

[package]
name = "vmm-cli"
version = "0.1.0"
edition = "2024"

[dependencies]
anyhow = { workspace = true }
vmm-core = { path = "../vmm-core" }
vmm-net = { path = "../vmm-net" }
```

This gives us:

```text
                 vmm-cli
                /       \
               v         v
          vmm-core     vmm-net
               \       /
                v     v
             shared dependencies
```

---

# 28. Workspace Resolver

The workspace can specify Cargo's dependency resolver:

```toml
[workspace]
resolver = "3"
```

The resolver determines how Cargo resolves dependencies and features across the dependency graph.

For modern Rust projects, explicitly specifying the resolver in the workspace root makes the intended workspace configuration clear.

Example:

```toml
[workspace]
resolver = "3"
members = [
    "crates/core",
    "crates/network",
]
```

---

# 29. Workspace Profiles

Build profiles such as release configuration are controlled from the workspace root.

For example:

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

This applies to the workspace's build profile.

A useful principle is:

```text
workspace root
    |
    +-- dependency resolution
    +-- package metadata
    +-- build profiles
    +-- workspace membership
```

Member manifests primarily describe the individual package.

---

# 30. Workspace `[patch]`

The workspace root is also where `[patch]` overrides are configured.

For example:

```toml
[patch.crates-io]
my-library = {
    git = "https://github.com/example/my-library"
}
```

This can be useful when testing a local or Git version of a dependency.

The important point is that `[patch]` is a workspace-root configuration.

---

# 31. Workspace `Cargo.toml` as Project Configuration

A useful mental model is:

```text
workspace Cargo.toml
    |
    +-- Which packages?
    |      members
    |
    +-- Which packages are default?
    |      default-members
    |
    +-- Shared dependency versions?
    |      workspace.dependencies
    |
    +-- Shared package metadata?
    |      workspace.package
    |
    +-- Dependency overrides?
    |      patch
    |
    +-- Build configuration?
           profile
```

Each member's `Cargo.toml` then describes:

```text
individual package
    |
    +-- package name
    +-- package-specific dependencies
    +-- targets
    +-- features
    +-- package-specific configuration
```

---

# 32. Common Workspace Structure

A common structure for a larger Rust project is:

```text
project/
├── Cargo.toml
├── Cargo.lock
├── README.md
│
├── crates/
│   ├── core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── storage/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── network/
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   └── api/
│       ├── Cargo.toml
│       └── src/
│
└── tools/
    └── cli/
        ├── Cargo.toml
        └── src/
```

Root:

```toml
[workspace]
resolver = "3"

members = [
    "crates/*",
    "tools/*",
]

[workspace.dependencies]
anyhow = "1"
thiserror = "2"
tokio = "1"
serde = "1"
```

---

# 33. Workspace and Crate Independence

A workspace does not merge crates together.

For example:

```text
core
network
cli
```

remain separate crates.

`network` cannot use a type from `core` simply because they are in the same workspace.

It needs an explicit dependency:

```toml
[dependencies]
core = { path = "../core" }
```

Then Rust code can use it:

```rust
use core::SomeType;
```

Workspace membership and crate dependencies are therefore separate concepts.

---

# 34. Workspace vs Module

Don't confuse a workspace with a Rust module.

A module:

```rust
mod network;
```

is part of a crate's source-code organization.

A workspace:

```toml
[workspace]
members = [
    "network",
]
```

is Cargo's project/package organization.

Think:

```text
Cargo level:

Workspace
    |
    +-- Package
          |
          +-- Crates
                |
                +-- Modules
                      |
                      +-- Functions / Types
```

---

# 35. Workspace vs Library

A library crate:

```bash
cargo new --lib core
```

creates a library package.

A workspace:

```toml
[workspace]
members = ["core"]
```

manages that package together with other packages.

Therefore:

```text
library != workspace
```

A workspace can contain:

```text
libraries
binaries
build tools
examples
tests
```

depending on how the project is organized.

---

# 36. Creating a Workspace

Start with:

```bash
mkdir my-project
cd my-project
```

Create the root manifest:

```toml
[workspace]
resolver = "3"

members = [
    "core",
    "cli",
]
```

Create the packages:

```bash
cargo new core --lib
cargo new cli
```

The resulting structure:

```text
my-project/
├── Cargo.toml
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
│
└── cli/
    ├── Cargo.toml
    └── src/main.rs
```

---

# 37. Adding an Internal Dependency

Suppose `cli` uses `core`.

In:

```text
cli/Cargo.toml
```

add:

```toml
[dependencies]
core = { path = "../core" }
```

Then:

```rust
use core::some_function;
```

The dependency is now part of the crate dependency graph.

---

# 38. Building the Workspace

From the workspace root:

```bash
cargo build --workspace
```

Cargo builds all selected workspace packages.

For development:

```bash
cargo check --workspace
```

is often faster because it checks the code without producing the final binaries.

---

# 39. Testing a Workspace

Run all tests:

```bash
cargo test --workspace
```

Run one package:

```bash
cargo test -p core
```

Run tests repeatedly while developing with an external tool such as `cargo-watch` if desired.

---

# 40. Inspecting the Workspace

Useful Cargo commands include:

```bash
cargo metadata
```

This is particularly useful when working with large workspaces because it exposes Cargo's package and dependency metadata in machine-readable form.

For example:

```bash
cargo metadata --format-version 1
```

can show:

```text
packages
workspace_members
workspace_root
target_directory
dependencies
```

This is useful for tooling and CI systems.

---

# 41. Workspace Commands Cheat Sheet

```bash
# Check all members
cargo check --workspace

# Build all members
cargo build --workspace

# Release build
cargo build --workspace --release

# Test all members
cargo test --workspace

# Run clippy
cargo clippy --workspace

# Build one package
cargo build -p my-crate

# Test one package
cargo test -p my-crate

# Run one binary package
cargo run -p my-cli

# Inspect workspace metadata
cargo metadata --format-version 1
```

---

# 42. Common Mistakes

## Mistake 1: Assuming workspace dependencies are automatically available

This:

```toml
[workspace.dependencies]
serde = "1"
```

does not automatically make `serde` available to every crate.

You still need:

```toml
[dependencies]
serde = { workspace = true }
```

---

## Mistake 2: Forgetting a member

If a package isn't included through `members` or another workspace membership mechanism, Cargo may not treat it as part of the workspace.

Check:

```bash
cargo metadata --format-version 1
```

---

## Mistake 3: Confusing path dependency with workspace membership

This:

```toml
core = { path = "../core" }
```

describes a dependency.

This:

```toml
[workspace]
members = ["core"]
```

describes workspace membership.

They solve different problems.

---

## Mistake 4: Putting workspace configuration in the wrong manifest

Some settings belong at the workspace root.

For example:

```toml
[workspace.dependencies]
```

belongs in the workspace root.

Likewise, workspace-level `[patch]` and `[profile]` configuration belongs in the root manifest.

---

# 43. Workspace Design Principle

A good workspace usually has a clear dependency direction.

For example:

```text
                 cli
                  |
                  v
                api
               /   \
              v     v
          network  storage
              \     /
               v   v
                core
```

Avoid unnecessary dependencies in the opposite direction:

```text
core
  |
  v
cli
```

because low-level crates should generally not depend on high-level application crates.

A workspace makes it easy to create many crates, but it does not automatically create good architecture.

The dependency graph still needs to be designed carefully.

---

# 44. Workspace in Large Rust Projects

Workspaces become particularly valuable when a project contains:

* multiple libraries
* multiple binaries
* reusable internal crates
* platform-specific components
* test utilities
* command-line tools
* benchmarks
* procedural macros
* system-level components

For example:

```text
rust-vmm/
├── vmm-core/
├── vmm-memory/
├── vmm-device/
├── vmm-net/
├── vmm-block/
└── tools/
```

Each crate can evolve independently while Cargo provides a unified project structure.

---

# 45. Workspace Mental Model

The most useful mental model is:

```text
                    WORKSPACE
                        |
        +---------------+---------------+
        |               |               |
      core           network           cli
        |               |               |
        +-------+-------+-------+-------+
                |
          dependency graph
                |
        +-------+-------+
        |               |
     external        internal
   dependencies     dependencies
```

The workspace is the **container for the project**.

The crates remain independent.

Cargo uses the workspace to coordinate:

```text
dependency resolution
Cargo.lock
build artifacts
package selection
shared metadata
shared dependencies
profiles
workspace-wide commands
```

---

# 46. Key Takeaways

* A **Cargo workspace** is a collection of packages managed together.
* A workspace can contain many independent crates.
* A workspace does not merge crates into one crate.
* Each member still has its own `Cargo.toml`.
* A **virtual workspace** has a root `Cargo.toml` without `[package]`.
* `members` defines workspace members.
* `exclude` can remove packages from workspace membership.
* `default-members` controls the default package selection from the workspace root.
* Workspace members normally share one `Cargo.lock`.
* Workspace members normally share one `target` directory.
* `[workspace.dependencies]` centralizes dependency declarations.
* `workspace = true` lets individual crates inherit workspace dependencies.
* Workspace dependencies are **not automatically available** to every crate.
* `[workspace.package]` can centralize package metadata such as version, edition, license, and repository.
* `-p` selects a particular package.
* `--workspace` selects all workspace members.
* `path = "../crate"` creates an internal path dependency; workspace membership is a separate concept.
* Workspace-level `[profile]` and `[patch]` configuration belongs in the workspace root.
* `cargo metadata` is useful for understanding and tooling around large workspaces.
* A workspace is primarily a **Cargo/project organization mechanism**, while modules organize code inside a crate.

The simplest way to remember it is:

```text
Workspace
    ↓
manages multiple packages

Package
    ↓
contains one or more crates

Crate
    ↓
contains Rust code

Dependency
    ↓
connects crates/packages together
```

For large Rust projects, the workspace is the layer that turns a collection of independent crates into a single manageable repository.
