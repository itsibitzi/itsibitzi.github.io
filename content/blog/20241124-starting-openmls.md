+++
title = "Creating a complete end-to-end encrypted chat app using OpenMLS, in Rust"
date = 2024-11-24
+++

Hey there, I'm Sam.

After several false starts trying to develop on top of OpenMLS I decided a good way to motivate myself to create a full application
was to turn it into a tutorial. Th

I'm writing this tutorial because I found the official OpenMLS book lacking in detail in a few key areas, and also so that I am
more motivated to learn OpenMLS myself.

## Disclaimer

These posts are for a fairly niche group; people who want to implement their own end-to-end encrypted applications on top of MLS.
In order for this to be a smooth experience for those folks I'm going to skip over some fundementals listed here:

- Rust development, to an intermediate level
- Asymetric cryptographic primatives
# TODO add more as we come across them

## Project Description

Throughout this tutorial we will be creating an encrypted chat application. It's not exactly the most creative choice for
working with MLS but hopefully everyone reading this should be familiar with the kinds of thing you'd expect out of an encrypted chat
client. Perhaps less conventionally, we will be creating this application as a command-line app. This is purely so that we don't end
up getting distracted creating a pretty user interface. We will use Rust's `clap` library to create a basic set of commands which users
can call in order to perform the basic operations.

- Register an account
- Add friends
- Create, view, and update groups
- Send messages
- Update your username and status

Rather uncreatively I've named the tool `cipher-chat`, but you're free to call yours whatever you want, just change the names accordingly.

## Getting Started

Before we dig into MLS we need to create the scaffolding of our project.

### Creating the workspace

Create a directory with the name `cipher-chat` (or whatever you'd like to call it) and add the following workspace `Cargo.toml`.

```toml
[workspace]
resolver = "2"

[workspace.dependencies]
# For simplicity I'm going to use the eyre library for everything
color-eyre = "0.6.3"

# Our client CLI wil be set up with the excellent clap crate.
clap = { version = "4.5.21", features = ["derive"] }

# We're going to use tokio with axum in our server and reqwest
# in our client to send messages, which will be serialized as JSON
tokio = { version = "1.41.1", features = ["full"] }
axum = "0.7.9"
reqwest = "0.12.9"
serde = { version = "1.0.215", features = ["derive"] }
serde_json = "1.0.113"
```

Now we're going to create three workspace crates:
- `client` which will be the command-line chat tool
- `server` which will act as both the authentication and delivery service.
- `common` for shared components

Let's create those now:

```shell
cargo new --lib common && cargo new client && cargo new server
```

This should add the new crates to the `workspace.members` section in your workspace Cargo.toml.

## Creating the basic client
