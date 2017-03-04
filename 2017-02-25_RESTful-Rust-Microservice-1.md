# Part 1 - Getting started

So I'm starting my own tech blog, and the first topic I'd like to cover is Rust, or more specifically, how I go about building a RESTful microservice in Rust. Bear in mind that this is not a tutorial, it's me telling the story of how I went about it what I think about the result.

I'll name this PoC project rest-battleship, because a lot of my experiments with the Rust language have been about this classic game, for reasons. The complete code can be found on [GitHub](https://github.com/Leopard2A5/rest-battleship).

## What's to be done?

The minimum requirements I have for this PoC are
* A JSON API
* REST- and meaningful responses, i.e. using appropriate HTTP response codes
* Database interaction

Part 1 will cover setting up the project and getting a minimal HTTP service up and running.

## Setting up the project

For this project I'll be using Rust 1.15.1, being the latest stable release at the time of writing. If you haven't got Rust installed yet, it's a breeze with [rustup](https://rustup.rs/). Version 1.15 has been a kind of milestone for Rust, as a long-awaited feature as become stable: *custom derive*. Custom derive allows you to create macros that can be used in Rust's `#[derive()]` attribute. This means you can finally generate custom code for structs on the stable branch of the language.

So `cargo new --bin rest-battleship` creates the project with a `Cargo.toml` file to describe the thing and a `src/main.rs` that will print 'Hello, world!' - easy!

## A minimal HTTP service

Okay, so the next step is to find a framework that let's us serve HTTP requests. There are several ones available and I've chosen `nickel` for this PoC. So let's add this dependency to `Cargo.toml`:
```
[package]
name = "rest-battleship"
version = "0.1.0"
authors = ["René Perschon <notmyemail@gmail.com>"]

[dependencies]
nickel = "0.9.0"
```

And then change `src/main.rs` to start a server and listen on port 8080:
```
#[macro_use] extern crate nickel;

use nickel::Nickel;
use nickel::HttpRouter;

fn main() {
    let mut server = Nickel::new();
    server.get("/games", middleware! {|_req, _resp|
        "Hello, world!"
    });
    server.listen("0.0.0.0:8080").expect("Error starting server");
}
```

Start the service with `cargo run` and that's enough to cURL [http://localhost:8080/games](http://localhost:8080/games) and receive a greeting to the whole world.

This concludes part 1 of the PoC. In [part 2](https://codingwithglee.blogspot.de/2017/03/my-shot-at-restful-microservices-in.html) I will cover basic database interaction.
