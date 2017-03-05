# Part 3 - Linking REST endpoint and db layer

Welcome to part 3 of my Rust microservices series! If you haven't read parts 1 or 2, here are the respective links: [part 1](http://codingwithglee.blogspot.com/2017/02/my-shot-at-restful-microservices-in.html) [part 2](https://codingwithglee.blogspot.de/2017/02/my-shot-at-restful-microservices-in.html). In this installment I'm going to connect the REST endpoint with the database layer and take care of serialization and deserialization of the Rust structs.

## JSON serialization

There are several crates that give you automatic serialization and deserialization of structs to JSON strings. I'm going to use Serde in this PoC. Serde is divided into a core crate and one additional crate per source/target format. So I'm going to use the crates `serde`, `serde_derive` and `serde_json`. The crate `serde_derive` contains the `Serialize` and `Deserialize` macros that implement the trais with same names. This enables us to serialize a struct by calling serde_json::to_string.

src/models/game.rs:
```rust
#[derive(Debug, Serialize)]
pub struct DbGame { /* omitted. */ }

#[derive(Debug, Serialize)]
pub struct Dimensions { /* omitted. */ }
```

src/main.rs:
```rust
#[macro_use] extern crate serde_derive;
extern crate serde_json;

fn main() {
    for game in dao::get_games() {
        println!("{:?}", serde_json::to_string(&game).unwrap());
    }
}
```

Unsurprisingly, deserialization works the same way.

## Connecting the REST endpoint to the database

I'm gonna create a simple endpoint listening on `GET /games` that will return a list of all games.
src/main.rs:
```
fn main() {
    let mut server = Nickel::new();
    server.get("/games", middleware! {|_req, mut resp|
        let games = dao::get_games();
        resp.set(MediaType::Json);
        serde_json::to_string(&games).unwrap()
    });
    server.listen("0.0.0.0:8080")
        .expect("Error starting server");
}
```

When I cURL this endpoint I get
```
< HTTP/1.1 200 OK
< Content-Type: application/json
< Date: Sun, 05 Mar 2017 18:26:36 GMT
< Server: Nickel
< Transfer-Encoding: chunked
<
* Connection #0 to host localhost left intact
[{"id":1,"dimensions":{"x":3,"y":3}},{"id":2,"dimensions":{"x":4,"y":5}}]
```
