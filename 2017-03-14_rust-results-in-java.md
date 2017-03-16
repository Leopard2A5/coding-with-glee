# Bringing Rust's Result type to Java

The Rust programming language was designed without exceptions to handle errors. Instead, the concept of errors is addressed with the generic `Result<T, E>` enum type. In this post I will compare Rust's and Java's error handling mechanisms and discuss if and how Rust's way of doing it can be applied to Java.

## Java exceptions

Java's error handling mechanism is built around the `Throwable` interface. Every type (i.e. interface, class) that is a subtype of `Throwable` can be thrown and caught in a try-catch block. The classes `Error` and `Exception` are subtypes of `Throwable` and whether a type inherits from `Error` or `Exception` is the first main distinction: `Errors` are really exceptional (no pun intended) situations that shouldn't be caught or handled by the overwhelming majority of programs. `Exceptions` on the other hand are the bread and butter for Java developers.

```
  +-----------+
  | Throwable |-------+
  +-----------+       |
        |             |
        V             V
    +-------+   +-----------+
    | Error |   | Exception |-----------+
    +-------+   +-----------+           |
                      |                 |
                      V                 V
          +------------------+   +-------------+
          | RuntimeException |   | IOException |
          +------------------+   +-------------+
```

When Java was designed, the decision was made to add the concept of *checked exceptions*. A checked exception is any class that inherits from `Exception` and doesn't also have `RuntimeException` as an ancestor, which in turn is a direct subtype of `Exception`. Whenever your code calls a method that may throw a checked exception, you **have** to handle it, either by adding a compatible *throws* declaration to your method or by catching it in a try-catch block.

## Rust Results

Rust's error handling mechanism is built around the generic `Result<T, E>` enum type. The enum is defined as follows:
```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```
A function that returns a string but can fail (recoverably) will define its return type as `Result<String, E>` where *E* is the type that is returned in the error case. Rust's type system is designed in such a way that the simplest definition of a Result is actually `Result<(), ()>`, so both the happy and the error case contain an empty tuple as the payload. Even this case conveys a minimal amount of information: whether the operation succeeded or failed, and it does so in a more explicit way than a boolean return type could achieve.

Similar to Java's `Error` and `Exception` types, Rust has a second way of 'handling' errors: the *thread panic*. When a thread panics it means something has gone horribly wrong. Thread panics are not meant to be caught, although there is a way supposedly. 
