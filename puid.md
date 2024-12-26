# PUID

I recently developed my first Rust crate, a library for generating [unique IDs](https://en.wikipedia.org/wiki/Universally_unique_identifier). The idea was inspired by [Twitter's snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake) and I was looking to bring a similar solution to Rust projects.

One of the key requirements I had in mind while creating this library was the ability to add a prefix to the IDs. Although unique IDs are not typically human-readable, adding a prefix can be helpful for debugging and for quickly identifying the objects associated with the IDs.

With that in mind, selecting a name for the crate was a no-brainer. I chose "puid," which stands for "prefix unique ID" :)

[PUID](https://crates.io/crates/puid) has a similar structure to Twitter snowflake, I made up the id of the following parts:

1. An 8-character prefix is chosen by the user.

```rust
impl<'a> PuidBuilder<'a> {
    // [...]

    pub fn prefix(self, prefix: &'a str) -> PuidResult<Self> {
        if validate(prefix) {
            Ok(Self { prefix, ..self })
        } else {
            Err(PuidError::InvalidPrefix)
        }
    }
}

const PREFIX_MAX_LEN: usize = 8;
const PREFIX_MIN_LEN: usize = 1;

// [...]

fn validate(prefix: &str) -> bool {
    (PREFIX_MIN_LEN..=PREFIX_MAX_LEN).contains(&prefix.len())
        && prefix.chars().all(|c| c.is_ascii_alphanumeric())
}
```

2. An underscore `_` between the prefix and the id characters.

```rust
impl<'a> PuidBuilder<'a> {
    // [...]

    pub fn build(self) -> PuidResult<String> {
        let mut result =
            String::with_capacity(self.prefix.len() + 1 + 16 + 3 + 16 + self.entropy as usize);

        result.push_str(self.prefix);
        result.push('_');
        result.push_str(&to_base36(time()));
        result.push_str(&counter().to_string());
        result.push_str(&to_base36(u128::from(std::process::id())));
        result.push_str(&rnd_string(self.entropy));

        // [...]
    }
}
```

3. The Unix timestamp.

```rust
fn time() -> u128 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_millis()
}
```

4. An incrementing counter that starts again at a limit of 255.

```rust
fn counter() -> u8 {
    COUNTER
        .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |i| match i {
            i if i == u8::MAX => Some(0),
            _ => Some(i + 1),
        })
        .unwrap()
}
```

5. The process identifier.

```rust
std::process::id() as u128
```

6. An alphanumeric sequence using Rand as a dependency.

```rust
fn rnd_string(elements: u8) -> String {
    thread_rng()
        .sample_iter(&Alphanumeric)
        .take(elements as usize)
        .map(char::from)
        .collect()
}
```

Parts 3 and 5, the Unix timestamp and the process identifier. These values are converted to [Base36](https://en.wikipedia.org/wiki/Base36) for compact representation. [Base36](https://en.wikipedia.org/wiki/Base36) is a compact and efficient way of representing numerical values using a combination of the digits 0-9 and the letters A-Z. This makes it a good choice for situations where space is limited.

```rust
fn to_base36(mut v: u128) -> String {
    // 16 characters cover most cases which is typical for base-36 encoding of a u128
    let mut result = String::with_capacity(16);
    while v > 0 {
        result.push(
            char::from_digit(
                u32::try_from(v % u128::from(BASE_36)).unwrap(),
                u32::from(BASE_36),
            )
            .unwrap(),
        );
        v /= u128::from(BASE_36);
    }
    result.chars().rev().collect()
}
```

The [PUID](https://crates.io/crates/puid) crate implements a builder pattern, providing a publicly accessible builder through the Puid struct. This allows customization of the PUID output. For example:

An id with a prefix `foo_` and default entropy of 12 random characters at the end.

```rust
let id = Puid::builder().prefix("foo")?.build()?;
```

An id with a prefix `bar_` and custom entropy of 24 random characters at the end.

```rust
let id = Puid::builder().prefix("bar")?.entropy(24).build()?;
```

And that's a wrap! If you want to know more, just try the [crate](https://crates.io/crates/puid) or hit up the [docs](https://docs.rs/puid/0.2.0/puid/) and [source code](https://github.com/aitorfernandez/puid).
