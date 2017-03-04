# Part 2 - Database interaction

Welcome back! If you haven't read part 1 yet: this series of blog posts is about
creating a simple RESTful service in Rust. After setting up the project in
[part 1](http://codingwithglee.blogspot.com/2017/02/my-shot-at-restful-microservices-in.html)
, I'm gonna set up a basic database interaction, to make the scenario
more realistic.

I initially wanted to use a full-fledged ORM solution for this PoC but then
decided it's better to concentrate on a few things at a time. To put it in a
nutshell, for this project I use [Diesel](http://diesel.rs)'s migration features
without the actual OR-mapping.


## Diesel setup

Diesel comes as a library and additionally as a tool for the command line,
called `diesel_cli`. I install the command line tool with
`cargo install diesel_cli`.

For diesel to know how to connect to the database I add a `.env` file to the
project:
```
DATABASE_URL=postgres://postgres@localhost/battleship
```
The `.env` file is just a means of collecting environment variables and it can
easily incorporated into your program with the `dotenv` library.

Now i need to create a database. I chose to just spin up a dockerized
postgres server for development purposes like so:
```
docker run \
  -d --name battleship_db \
  -p 5432:5432 -e POSTGRES_PASSWORD='' \
  postgres
```

When I now run `diesel setup` two things happen:
1. a `migrations` directory is created
1. the battleship database is created inside the postgres container


## A database migration

Now that there is a database, I'll create a migration to initialize the
database with a table. I run `diesel migration generate create_games`, which
creates two files in `migrations/20170301195954_create_games/`: `up.sql` and
`down.sql`. Unsurprisingly, one of them is used to make a change in the
database, whereas the other reverts the change.

up.sql:
```sql
CREATE TABLE games (
  id BIGSERIAL NOT NULL PRIMARY KEY,
  dimension_x INTEGER NOT NULL,
  dimension_y INTEGER NOT NULL
);
```

down.sql:
```sql
DROP TABLE games;
```

I now run `diesel migration run` and up.sql is executed in the dockerized
database.


## The model

I need a representation of a game in Rust, so I create the following structs:

src/models/game.rs:
```rust
#[derive(Debug)]
pub struct DbGame {
    pub id: i64,
    pub dimensions: Dimensions,
}

#[derive(Debug)]
pub struct Dimensions {
    pub x: i32,
    pub y: i32,
}
```


## Interacting with the db

Since I'm not using an OR-Mapper, I'm gonna query the database through plain
SQL, using the native postgres driver (added to `Cargo.toml`). Futhermore, I'll
use `dotenv` to get the database connection URL from `.env`.

I create a method that establishes a database connection and another one that
queries the games table for all entries. The latter iterates over the results
and maps each row to a `DbGame` using one of the standard type conversion
mechanisms in Rust, the `From` trait. For this to work, there must be an
implementation of `From<Row>` for `DbGame`, which is listed below.

src/dao/game_dao.rs:
```rust
use dotenv::dotenv;
use models::DbGame;
use postgres::{Connection, TlsMode};
use std::env;

fn connect() -> Connection {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    Connection::connect(&*database_url, TlsMode::None)
        .expect(&format!("Error connecting to {}", &database_url))
}

pub fn get_games() -> Vec<DbGame> {
    let conn = connect();
    let rows = conn.query("SELECT * FROM games", &[])
        .expect("Error querying database");

    rows.iter()
        .map(DbGame::from)
        .collect()
}
```

src/models/game.rs:
```rust
impl<'a> From<Row<'a>> for DbGame {
    fn from(row: Row) -> Self {
        DbGame {
            id: row.get("id"),
            dimensions: Dimensions {
                x: row.get("dimension_x"),
                y: row.get("dimension_y"),
            },
        }
    }
}
```

I can then list the database entries in `main.rs`:
```
fn main() {
    for game in dao::get_games() {
        println!("{:?}", game);
    }
}
```
Which yields the following output for me, after I've manually inserted some
data:
```
DbGame { id: 1, dimensions: Dimensions { x: 3, y: 3 } }
DbGame { id: 2, dimensions: Dimensions { x: 4, y: 5 } }
```

This concludes part 2 of the PoC. In part 3 I will show how I connected the
database layer with the REST endpoint and how to convert the Rust structs into
JSON.
