+++
title = "Rust template with workspace and logging"
weight = 1
order = 1
date = 2023-03-27
insert_anchor_links = "right"
[taxonomies]
tags = ["rust", "setup", "tldr"]
+++

The repository can be found there: [https://github.com/aurelien-clu/template-rust](https://github.com/aurelien-clu/template-rust)

## Target

- have a rust project in which we can have multiple libraries & multiple binaries
- logging should be setup for each crate (library or binary)

### Layout

```
.
├── Cargo.lock
├── Cargo.toml
├── README.md
└── crates
    ├── bin-client
    │   ├── Cargo.toml
    │   └── src
    │       └── main.rs
    ├── bin-server
    │   ├── Cargo.toml
    │   └── src
    │       └── main.rs
    └── domain
        ├── Cargo.toml
        └── src
            └── lib.rs
```

## How to reuse

```bash
git clone https://github.com/aurelien-clu/template-rust <your-project>
```

## How to reproduce the template

### .gitignore

Let's create the `.gitignore` file with:

```bash
echo target > .gitignore
```

### Cargo.toml

Let's create the `Cargo.toml` file with:

```toml
[workspace]

members = ["crates/*"]

[workspace.package]
version = "0.1.0"
authors = ["Aurélien Clu. <john@doe.johndoe>"]
description = "template-rust"
readme = "README.md"
edition = "2021"
license = "UNLICENSE"
```

### crates/

Now we add the crates we want to have:

```bash
cargo new crates/bin-client
cargo new crates/bin-server
cargo new crates/domain --lib
# feel free to add more libraries
```

And update their configuration (Cargo.toml) to have a proper name, inherit fields from the workspace configuration and have some useful default dependencies.

#### crates/bin-client/Cargo.toml

```toml
[package]
name = "client"

version.workspace = true
authors.workspace = true
description.workspace = true
readme.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
domain = { path = "../domain" }

# logging
log = "^0"
pretty_env_logger = "^0"
```

#### crates/bin-server/Cargo.toml

```toml
[package]
name = "server"

version.workspace = true
authors.workspace = true
description.workspace = true
readme.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
domain = { path = "../domain" }

# logging
tracing = "^0"
tracing-log = "^0"  # to handle logs from log, e.g. from our domain library here
tracing-subscriber = "^0"  # to output to stdout

# error handling
thiserror = "^1"
```

#### crates/domain/Cargo.toml

```toml
[package]
name = "domain"

version.workspace = true
authors.workspace = true
description.workspace = true
readme.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
# logging
log = "^0"

# error handling
thiserror = "^1"
```

### Run

```bash
cargo test

cargo run --bin server
cargo run --bin client

# or building and then running
cargo build --release --bin server
cargo build --release --bin client
./target/release/server
RUST_LOG=TRACE ./target/release/client
```

To print logs, feel free to update your files alike:

- [crates/bin-client/src/main.rs](https://github.com/aurelien-clu/template-rust/blob/main/crates/bin-client/src/main.rs)
- [crates/bin-server/src/main.rs](https://github.com/aurelien-clu/template-rust/blob/main/crates/bin-server/src/main.rs)
- [crates/domain/src/lib.rs](https://github.com/aurelien-clu/template-rust/blob/main/crates/domain/src/lib.rs)

And running again should give the following:

<img src="https://raw.githubusercontent.com/aurelien-clu/template-rust/main/docs/terminal.png" alt= "terminal" width="75%" height="75%"/>
