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
        resp.set(MediaType::Json);
        let games = dao::get_games();
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

## Deserializing JSON

So now that we've got a working endpoint that lists all the games, let's add one that actually creates a game. I'm gonna keep things simple here and let the caller choose the id of the game and not care about key uniqueness issues for this PoC. The first step is to add the `Deserialize` macro to the entity structs. After that it's mostly about the dao and the controller code.

src/main.rs:
```rust
// ...
server.post("/games", middleware! {|req, mut resp|
    match get_game_from_request(req) {
        Ok(game) => {
            resp.set(StatusCode::Created);
            dao::create_game(game);
            "Ok!".to_string()
        },
        Err(e) => {
            resp.set(StatusCode::BadRequest);
            e
        }
    }
});
// ...
fn get_game_from_request(
    req: &mut nickel::Request,
) -> Result<DbGame, String> {
    let mut body = String::new();
    req.origin.read_to_string(&mut body).unwrap();
    serde_json::from_str::<DbGame>(&body)
        .map_err(|e| e.description().to_string() )
}
```

Nickel provides built-in JSON deserialization, but this feature relies on the `rustc_serialize` crate, which I'm not using. Serde is a newer and more modular implementation for serialization and deserialization. The `get_game_from_request` function extracts the body from the request and then tries to deserialize it. The database access code is straight-forward:

game_dao.rs:
```rust
pub fn create_game(game: DbGame) {
    let conn = connect();
    conn.execute(r#"
        INSERT INTO games (id, dimension_x, dimension_y)
        VALUES ($1, $2, $3)"#,
        &[&game.id, &game.dimensions.x, &game.dimensions.y]
    ).expect("Error inserting into database");
}
```

As promised at the beginning, I don't care about primary key uniqueness in this PoC, so if you try to POST a game with an id that's already there, the thread is going to panic.

## Conclusions

We've seen that it is possible to create microservices in Rust with little effort, even though compared to older languages there's more boilerplate code that you have to write yourself. Especially Nickel seems to have a lot of room for improvement. I don't like that you seem to have to return a String from every endpoint definition in the `middleware!` macro, but then I'm not very good at reading macro definitions in Rust yet.

One could think that interacting with postgres directly and not using an OR-Mapper is a bad idea, but I think that especially in microservices, the number of entities is usually small enough for that not to matter too much.

This concludes the third and last part of this proof of concept. You can find the source code [here](https://github.com/Leopard2A5/cwg-rest-battleship). Thanks for reading and please feel free to comment.
