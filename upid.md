# UPID

I've been thinking over the [crate](https://crates.io/crates/puid) I created some time ago and considering ways to modify it. The crate employs a builder pattern to generate IDs. While the builder pattern enhances readability and allows for setting only the necessary fields, among other advantages, I'm starting to think that, in the context of generating an ID, it might be excessive. Echoing the sentiments of John Carmack:

> Sometimes, the elegant implementation is just a function. Not a method. Not a class. Not a framework. Just a function.

The current builder is structured like this:

```rs
pub struct PuidBuilder<'a> {
    entropy: usize,
    prefix: &'a str,
}

impl<'a> PuidBuilder<'a> {
    pub fn new() -> Self {
        Self {
            entropy: 12,
            ..Self::default()
        }
    }

    // [...]

    pub fn build(self) -> PuidResult<String> {
        // [...]
    }
}
```

Instead of a struct, replace it with a function, and increasing the limitation of the prefix from 8 characters to 12 characters:

```rs
const MAX_PREFIX_LEN: usize = 12;

// [...]

pub fn upid(pref: &str) -> String {
    assert!(
        validate(pref),
        "Prefix exceeds maximum allowed length. {{ {MAX_PREFIX_LEN} }} or is not alphanumeric"
    );

    // [...]
}
```

The `assert!` macro will trigger a panic with the specified error message. Although the builder solution allows the user to handle errors and perhaps prepare a fallback solution, I believe it's better to panic than to proceed without an ID in this scenario.

It's good to add a `# Panics` section for functions that can panic:

```rs
/// # Panics
///
/// Will panic if prefix is not asscii alphanumeric or bigger than `MAX_PREFIX_LEN`
pub fn upid(pref: &str) -> String {
    // [...]
}
```

Another change I've been considering is converting the time into hexadecimal instead of base36. Hexadecimal is easy to parse and more compact than base36:

```rs
/// Returns the total number of whole milliseconds from Unix epoch.
fn millis() -> u128 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("System time is before UNIX epoch")
        .as_millis()
}

fn time_to_hex() -> String {
    let milliseconds = millis();

    let seconds = milliseconds / MILLISECONDS_PER_SECOND;
    let milliseconds = milliseconds % MILLISECONDS_PER_SECOND;
}
```

After these changes, my first thought is to return a string using the `format!` macro:

```rs
fn time_to_hex() -> String {
    // [...]
    format!("{seconds:08x}-{milliseconds:04x}")
}
```

But using the `write!` from the `std::fmt` module avoids the overhead of allocating memory for the formatted string and is a bit faster:

```rs
use std::fmt::Write;

// 8 for seconds + 1 for hyphen + 4 for milliseconds
const TIME_LEN: usize = 13;

// [...]

fn time_to_hex() -> String {
    // [...]

    let mut result = String::with_capacity(TIME_LEN);
    write!(result, "{seconds:08x}-{milliseconds:04x}")
        .expect("Failed to write the formatted string");
    result
}
```

```sh
test bench_format ... bench: 690 ns/iter (+/- 57)
test bench_write ... bench: 674 ns/iter (+/- 66)
```

Another change after removing the builder is maintaining a fixed entropy. Currently, the crate defaults to 12, but users can customize it:

```rs
impl<'a> PuidBuilder<'a> {
    // [...]

    /// Add the custom entropy.
    pub fn entropy(self, entropy: usize) -> Self {
        Self { entropy, ..self }
    }

    pub fn build(self) -> PuidResult<String> {
        if self.prefix.is_empty() {
            Err(PuidError::InvalidPrefix)
        } else {
            Ok([
                self.prefix,
                "_",
                &to_base36(time()),
                &counter().to_string(),
                &to_base36(std::process::id() as u128),
                &rnd_string(self.entropy),
            ]
            .concat())
        }
    }
}

fn rnd_string() -> String {
    thread_rng()
        .sample_iter(&Alphanumeric)
        .take(ENTROPY)
        .map(char::from)
        .collect()
}
```

Going with the idea of removing the builder, the best solution to keep it simple, is to go with a fixed entropy and keep the random part of the ID at 96 bits:

```rs
const ENTROPY: usize = 24;

// [...]

fn rnd_string() -> String {
    let mut rng = thread_rng();
    (0..ENTROPY)
        .map(|_| rng.sample(&Alphanumeric) as char)
        .collect()
}
```

In the end, just the function without builder as planned returning an ID.

```rs
const SEPARATOR: &str = "-";

// [...]

pub fn upid(pref: &str) -> String {
    // [...]

    [pref, SEPARATOR, &time_to_hex(), SEPARATOR, &rnd_string()].concat()
}
```

With these new changes, the parts of the UPID will follow as:

1. A maximum 12-character prefix chosen by the user.
2. A hyphen `-` between the prefix and hexadecimal time.
3. The Unix timestamp as hexadecimal with the format seconds-milliseconds.
4. A hyphen `-` between the prefix and randomnes part.
5. An alphanumeric random sequence.
