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

Similar to Java's `Error` and `Exception` types, Rust has a second way of 'handling' errors: the *thread panic*. When a thread panics it means something has gone horribly wrong. Thread panics are not meant to be caught. Even though panics are are comparatively extreme measure, they're not without use.

When you design your API you have to think about what kinds of errors deserve to be treated as recoverable and what kinds are for example the result of false usage. An example from the Java world would be the well-known `NullPointerException`: You usually wouldn't define a catch block that handles an NPE, because it's usually the developer's fault. A `NumberFormatException` on the other hand could very well be the result of a user entering an invalid value.

An example for this kind of consideration in API design is Rust's `Vec` type: the `get` method returns an `Option<T>`, so trying to get an index that doesn't exist in the data structure will never panic, but it could tell you that there's nothing under that index by returning `None`. However, `Vec` also allows you to access its elements by index, using the square bracket syntax (`foo[5]`), but it's important to know that this will make the thread panic if the index is out of bounds. It's important to get this design right because it greatly influences the usability of your API - panic too often and the users of your API need to do a lot of verifications; overuse the `Result` type and developers need to handle them all over the place - in both cases the usability of your API suffers.

## A Result type for Java

I've created the [result-flow](https://github.com/Leopard2A5/result-flow) library, which brings a `Result` interface and the implementing classes `Ok` and `Err`. You can easily try it out by adding the following dependency to your project:
```xml
<dependency>
  <groupId>com.github.leopard2a5</groupId>
  <artifactId>result-flow</artifactId>
  <version>1.0.0</version>
</dependency>
```

Consider the following example:
```java
public class Numbers {
	public static void main(final String[] args) {
		final Result<Integer, String> result = readLine()
			.andThen(Numbers::parseInt)
			.map(Numbers::doubleUp);
		System.out.println(result);
	}

	private static Integer doubleUp(final Integer value) {
		return value * 2;
	}

	private static Result<Integer, String> parseInt(final String input) {
		try {
			return Result.ok(Integer.parseInt(input));
		} catch (final NumberFormatException e) {
			return Result.err(e.getMessage());
		}
	}

	private static Result<String, String> readLine() {
		try {
			final InputStreamReader in = new InputStreamReader(System.in);
			final BufferedReader buf = new BufferedReader(in);
			return Result.ok(buf.readLine());
		} catch (final IOException e) {
			return Result.err(e.getMessage());
		}
	}
}
```

The main method reads a line from *stdin*, then parses the read line to an Integer and finally doubles the value. As you can see, this code does not handle an error at all, it simply prints the result at the end. If the user enters a valid integer the output will be something like `Ok(14)`. Should the user input something like 'a', the output will be `Err(For input string: "a")`, so the Err wraps the message of the NumberFormatException.

Notice the difference between `andThen` and `map`: The former is used when the method to be called returns a Result, whereas the latter is used when that method does not fail with a Result itself.

Notice also that an IOException that occurs when we try to read from the InputStream will also be wrapped in an Err. This obviously doesn't make a lot of sense in production code. Depending on the context, an IOException would rather be treated as an exceptional or unrecoverable error.

Hence, my advice would be to keep any truly exceptional and/or unrecoverable errors like the aforementioned *panics* in Rust and use the conventional try-catch block on some level in the call stack. For errors of the application domain however, I think the pattern could be applicable on the JVM.
