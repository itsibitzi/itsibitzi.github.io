+++
title = "Creating a complete end-to-end encrypted chat app, in Rust, using OpenMLS"
date = 2024-11-24
+++

Hey there, I'm Sam.

After several false starts trying to develop on top of OpenMLS I decided a good way to motivate myself to create a full application
was to turn it into a tutorial.

So this is it.

## What is (Open)MLS

Message Layer Security (MLS) is a relatively new protocol aimed at providing a security layer for end-to-end encrypted (E2EE) applications.
It uses some pretty fancy data structures and algorithms in order to achieve a great number of security properties for group conversations
without compromising on performance. I've used the word "conversations" here, but the application doesn't need to be limited to just chat. You
can perhaps imagine many kinds of application that might want to share encrypted messages amongst a group of people. Examples might include an
E2EE document editor, where each "chat message" is actually an edit to a document, or a calendar app, where users can share their calendars
with groups of friends confidentially.

Futher reading:
https://messaginglayersecurity.rocks/mls-architecture/draft-ietf-mls-architecture.html

OpenMLS is an open source implementation of the MLS specification in the Rust programming language.

## Disclaimer

I provide no guarantee that any of the code produced in this tutorial is secure or fit for any purpose other than
learning about the OpenMLS library. I'm going to cut a lot of corners in order to keep this managable.

> I will flag when we come across a situation where I'm taking shortcuts using indented text like this.
> Some sections may include references to further reading or just food for thought, but like I say, I don't make
> any guarantees that the code in this article is secure or should be used for anything serious.

Please, for the sake of your users, do not copy any of this code into any real world application.

This tutorial is for a fairly niche group; people who want to implement their own end-to-end encrypted applications on top of MLS.
In order for this to (hopefully) be a smooth experience, I'm going to skip explaining quite a few concepts.

If you're already convinced that you need something like MLS, and would like to have somewhere to start with
OpenMLS specifically, then this blog post probably is for you.

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

# Let's go!

Before we dig into MLS we need to create the scaffolding of our project.

### Creating the workspace

Create a directory with the name `cipher-chat` (or whatever you'd like to call it) and add the following workspace `Cargo.toml`.

```toml,linenos
[workspace]
resolver = "2"

[workspace.dependencies]
# For simplicity I'm going to use the eyre library for everything
color-eyre = "0.6.3"

# Our client CLI wil be set up with the excellent clap crate.
clap = { version = "4.5.21", features = ["derive"] }

# We're going to use axum in our server and reqwest in our client to send
# messages, which will be serialized as JSON. Both clients and servers
# will store their data in SQLite databases
tokio = { version = "1.41.1", features = ["full"] }
axum = "0.7.9"
reqwest = "0.12.9"
serde = { version = "1.0.215", features = ["derive"] }
serde_json = "1.0.113"
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite"] }

```

I'm going to define every dependency in the workspace `Cargo.toml` and each package will import them using `crate-name.workspace = true`.

Now, from the workspace root, we're going to create three more crates:
- `client` which will be the command-line chat tool
- `server` which will be our API, from the MLS architecture document this will act as *both* the authentication and delivery services
- `common` for shared components such as forms to be sent from the client to the server.

Let's create those now with `cargo new --lib common && cargo new client && cargo new server`

This should add the new crates to the `workspace.members` section in your workspace Cargo.toml.

## Creating the client

Now that we have our workspace more or less sorted (we'll add some more stuff to it shortly) we can create our
client application.

To begin, we will just stub out all the commands that we will need. Add `clap.workspace = true` and `reqwest.workspace = true`
to `client/Cargo.toml` and create a new file: `client/src/cli.rs`.

Our `cli.rs` module will look like this:

```rust,linenos
use std::path::PathBuf;

use clap::{Parser, Subcommand};
use reqwest::Url;

#[derive(Parser)]
pub struct Cli {
    #[clap(long)]
    pub api_url: Url,
    #[clap(long)]
    pub user_db_path: PathBuf,
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

You can see we've created a new `Cli` structure with two required flags `api_url`, which is hopefully self-explainitory
and `user_db` which will be a path to SQLite database containing the users data.

Each user will have their own database and during testing it is how we will differentiate between which clients we're impersonating.
If that isn't clear right now it should become clearer later on, when we start testing our project.

> I'm not going to address at-rest encryption of the users database in this tutorial, since exactly how it's done would depend on your
> use case and threat model. In the past I've used SQLCipher, which is a popular open source extension to SQLite that transparently
> encrypts database pages as they are written to disk. It's easy to use and widely deployed in the real world. If you need some other
> security features (such as plausible deniability provided physical access to the user device) then you might need to come up with a
> more custom solution.

Now that we have some CLI arguments defined we need to update `main.rs` to parse those arguments and select an action based on the command.

```rust,linenos
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

As you can see, for now we're not handling loading the database, checking the API for liveness, or handling any kind of command.
We will fill in these capabilities as we go along.

Phew, so that's our basic client done. You can check to see if there are any mistakes now by running
`cargo run --bin client -- --help`. Your application should compile and print a help message. Running any of the other commands
right now would just result in a panic and `not yet implemented` being printed.

Time to go onto the server.

## Scaffolding the Server

We're going to structure the server crate in largely the same way as the client. First add `color-eyre`, `tokio`, `axum`, and `clap` as workspace
dependencies to `server/Cargo.toml` in the same fashion as before. Then we will get onto creating our CLI in `server/src/cli.rs`

```rust,linenos
use std::{net::SocketAddrV4, path::PathBuf};

use clap::Parser;

#[derive(Parser)]
pub struct Cli {
    pub db_path: PathBuf,
    pub listen_on: SocketAddrV4,
}

```

Our server CLI is a bit more straight forward.

Like with the `client`, we're going to use a SQLite database on the server, this time to store user details, encrypted messages, etc.

Now that we have the basic command line interface for the server defined, we can move onto updating it's `main.rs`.

```rust,linenos
use axum::{routing::get, Router};
use clap::Parser as _;
use cli::Cli;

mod cli;

#[tokio::main]
async fn main() -> color_eyre::Result<()> {
    color_eyre::install()?;

    let cli = Cli::parse();

    let app = Router::new().route("/", get(|| async { "hello cipher-chat!" }));

    let listener = tokio::net::TcpListener::bind(cli.listen_on).await?;

    axum::serve(listener, app).await?;

    Ok(())
}

```

There's a bit more going on here compared to the largely stubbed out `client`. We're parsing the `Cli` as before, but this time
we're also setting up a basic `axum` router. For now all it does is return a welcome string from it's root path, but we'll add more
functionality in due course. We've annotated our main function with `tokio::main` in order to make it `async` and we're returning
a `color-eyre` result.

You can test this now using `cargo run --bin server -- ./example.db 127.0.0.1:2222` and if all has gone to plan you should be able
to curl `localhost:2222` and see your message. To see the wonderful error messages `color-eyre` provides you can try to spin up another
server on the same port.

# Conclusion

Ooft. That was quite a lot. And we've not even started working with OpenMLS yet!

In the next exciting episode we will look at implementing our basic user objects and registrations.
