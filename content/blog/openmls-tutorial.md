+++
title = "Creating a complete end-to-end encrypted chat app, in Rust, using OpenMLS"
date = 2024-11-24
+++

Hey there, I'm Sam.

After several false starts trying to develop ontop of OpenMLS I decided a good way to motivate myself to create a full application
was to turn it into a tutorial.

So this is it.

## What is (Open)MLS

Message Layer Security (MLS) is a relatively new protocol aimed at providing a security layer for end-to-end encrypted (E2EE) applications.
It uses some pretty fancy data structures and algorithms in order to achieve a great number of security properties for group conversations
without compromising on performance. I've used the word "conversations" here, but the application doens't need to be limited to just chat. You
can perhaps imagine many kinds of application that might want to share encrypted messages amongst a group of people. Examples might include an
E2EE document editor, where each "chat message" is actually an edit to a document, or a calendar app, where users can share their calendars
with groups of friends confidentially.

OpenMLS is an open source implementation of the MLS specification in the Rust programming language.

## Disclaimer

I provide no guarantee that any of the code produced in this tutorial is secure or fit for any purpose other than
learning about the OpenMLS library. Please, for the sake of your users, do not copy any of this code into any real world application.

This tutorial is for a fairly niche group; people who want to implement their own end-to-end encrypted applications on top of MLS.
In order for this to (hopefully) be a smooth experience, I'm going to skip explaining quite a few concepts.

If you're already convinced that you need something like MLS, then this blog post probably is for you.

## Project Description

Throughout this tutorial we will be creating an encrypted chat application. It's not exactly the most creative choice for
working with MLS but hopefully everybody reading should be familiar with the kinds of thing you'd expect out of an encrypted chat
client. Perhaps less conventionally, we will be creating this application as a command-line app. This is purely so that we don't end
up getting caught in the weeds creating a user interface. We will use Rust's `clap` library to create a basic set of commands which users
can call in order to perform the basic operations.

- Register an account with a username and password.
- Add friends by their usernames
- Create, view, and update groups
- Send messages
- Update your username and status

Rather uncreatively, I've decided to name my tool `cipher-chat`, but you're free to call yours whatever you want, just change the names accordingly.

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

# We're going to use axum in our server and reqwest in our client to send
# messages, which will be serialized as JSON.
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

## Creating the client

Now that we have our workspace more or less sorted (we'll add some more stuff to it shortly) we can create our
client application.

To begin, we will just stub out all the commands that we will need. Add `clap.workspace = true` and `reqwest.workspace = true`
to `client/Cargo.toml` and create a new file: `client/src/cli.rs`.

Our `cli.rs` module will look like this:

```rust
use clap::{Parser, Subcommand};
use reqwest::Url;

#[derive(Parser)]
pub struct Cli {
    pub api_url: Url,
    #[clap(subcommand)]
    pub command: Command,
}

#[derive(Subcommand)]
pub enum Command {
    Register {},
    AddFriend {},
    CreateGroup {},
    ViewGroupMessages {},
    AddToGroup {},
    SendMessageToGroup {},
    UpdateStatus {},
}
```

And then we can update `main.rs` to parse those arguments and select an action based on the command.

```rust
use clap::Parser as _;
use cli::{Cli, Command};

mod cli;

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Command::Register {} => todo!(),
        Command::AddFriend {} => todo!(),
        Command::CreateGroup {} => todo!(),
        Command::ViewGroupMessages {} => todo!(),
        Command::AddToGroup {} => todo!(),
        Command::SendMessageToGroup {} => todo!(),
        Command::UpdateStatus {} => todo!(),
    }
}
```
