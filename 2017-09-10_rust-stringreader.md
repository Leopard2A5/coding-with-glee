I was recently working on a rust crate (read library) for parsing input files for Conway's game of life. The idea was to have a `Parser` trait like this:
```rust
use std::io::Read;

pub trait Parser {
	fn parse<T: Read>(&mut self, input: T) -> Result<GameDescriptor>;
}
```

The idea was to be as rustacenous (code like a rustacean?) as possible and use the standard traits where applicable. In this case this means accepting a `Read` as input, similarly to accepting an `InputStream` in a Java class.

I was writing unit tests when I came upon a slight problem: I didn't want to create test data files for simple tests, so ideally in my test I would just pass a string to the `parse` method. But Rust's `String` doesn't implement `Read` (why not??). Recalling the rules around trait implementations, I knew I couldn't implement `Read` for `String` myself.

The Rust [book](https://doc.rust-lang.org/book/first-edition/traits.html#rules-for-implementing-traits):
> either the trait or the type you’re implementing it for must be defined by you. Or more precisely, one of them must be defined in the same crate as the impl you're writing.

I didn't find any solution for this on the web, and no crates that help with this issue either. So I decided to publish one myself: [stringreader](https://crates.io/crates/stringreader).

## StringReader

Here's the basic code:
```rust
pub struct StringReader<'a> {
    iter: Iter<'a, u8>,
}

impl<'a> StringReader<'a> {
    pub fn new(data: &'a str) -> Self {
        Self {
            iter: data.as_bytes().iter(),
        }
    }
}
```
Simple enough, the struct contains an `Iter<'a, u8>` (more on that later), and there's a constructor that accepts a `&'a str`, so the lifetime of the `StringReader` must not exceed that of the input. The iterator inside `StringReader` is initialized by converting the input to a slice of bytes (`&[u8]`) and then getting an iterator for that slice.

## Implementing the Read trait
The `std::io::Read` trait requires only a single method to be implemented, but it provides some additional helpers via default implementations. In order to make `StringReader` implement `Read` one needs to provide an implementation for `read`:
```rust
fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
```

The read method accepts a mutable slice of `u8` (read unsigned byte) and returns a `std::io::Result<usize>`, where the positive result should contain the number of bytes read. At first I thought "why isn't there a parameter telling me how many bytes to read?", but then I remembered that slices in Rust know their length, so there can't be a buffer overflow.

My implementation for the trait looks like this:
```rust
impl<'a> Read for StringReader<'a> {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        for i in 0..buf.len() {
            if let Some(x) = self.iter.next() {
                buf[i] = *x;
            } else {
                return Ok(i);
            }
        }
        Ok(buf.len())
    }
}
```

I iterate over a range from 0 to the length (exclusive) of the passed-in byte slice. Then, for each index, I try to get the next byte from the iterator and, if it's a `Some`, write it to the slice. Otherwise, if the iterator returns a `None`, we know we've reached the end of the string, so we return a positive result with the number of bytes read. If the end of the for loop is reached we just return the length of the slice.

## Using StringReader in practice
Using the newly created crate, i could write relatively simple tests like this one:
```rust
use stringreader::StringReader;
// init parser
let input = StringReader::new("#N");
parser.parse(input);
// assertions
```

## Conclusions
I was really surprised that `std::io::Read` isn't implemented for Rust's string type(s), but in the end this allowed me to contribute a (imho) useful crate to the community. That Rust's slices provide an iterator was very helpful in accomplishing the task. All in all I'm satisfied with the crate and I hope it'll help others as well.

Thanks for reading!
