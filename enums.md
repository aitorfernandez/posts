# Rust Enums

Enumerations, or [enums](https://en.wikipedia.org/wiki/Enumerated_type) for short, are a set of predefined constants in a programming language. Each constant represents a specific value, and only one of these values can be active at any given time. A common use case of enumerations is to define a set of possible states, such as HTTP status codes.

```rs
enum HttpStatus {
    Success,
    Created,
    Accepted,
    NoContent,
    // ...
}
```

In Rust, [enums](https://doc.rust-lang.org/book/ch06-00-enums.html) are a versatile data type that can be used for more than just grouping related constants. The [Option](https://doc.rust-lang.org/std/option/enum.Option.html) type from the standard library, for example, is an enum that can either be `None` or `Some(T)`, where T is any type.

```rs
enum Option<T> {
    None,
    Some(T),
}
```

Enums, as previously mentioned, are useful for grouping similar types, like the example of `HttpStatus` codes. Another common example is error codes, where enums can provide a clear and organized way to define a set of possible errors for a given context, for example, database errors.

```rs
enum DatabaseError {
    PrivilegeNotGranted,
    NoData,
    ConnectionFailure,
    // ...
}
```

In the example of `DatabaseError` (1), the values have implicit discrimination. Rust will automatically assign numbers starting at 0 to each value in the enum.

```rs
assert_eq!(DatabaseError::PrivilegeNotGranted as i32, 0);
assert_eq!(DatabaseError::NoData as i32, 1);
assert_eq!(DatabaseError::ConnectionFailure as i32, 2);
```

We can use explicit discrimination to assign our values to the variants in an enum.

```rs
enum DatabaseError {
    PrivilegeNotGranted = 1007,
    NoData = 2000,
    ConnectionFailure = 8008,
    // ...
}

assert_eq!(DatabaseError::PrivilegeNotGranted as i32, 1007);
assert_eq!(DatabaseError::NoData as i32, 2000);
assert_eq!(DatabaseError::ConnectionFailure as i32, 8008);
```

Status and Errors are common examples of how enums can be used to group similar types, but let's explore a more complex and interesting use case.

In the video game [Diablo](<https://en.wikipedia.org/wiki/Diablo_(video_game)>), characters take damage as a result of fights, from a monster to a player or vice versa. A typical kind of enum for representing damage in the game could be:

```rs
enum Damage {
  Physical,
  Fire,
  Cold,
  Lightning,
  Poison,
}
```

As previously mentioned, in Rust enums are much more and an enum can contain any kind of data. We can modify the types of damage to permit store ranges of damage.

```rs
enum Damage {
    Physical { critical_hit: bool },  // 1
    Fire(i32),                        // 2
    Cold(i32, std::time::Duration),   // 3
    Lightning,                        // 4
    Poison(i32, std::time::Duration), // 5
}
```

We've updated the `Damage` enum.

The `Physical` type _(1)_ now includes an anonymous struct with a critical_hit `boolean` property.

The `Fire` type _(2)_ now accepts an `i32` integer type to represent the total damage dealt.

The `Cold` type _(3)_ includes two values, an `i32` integer type for total damage and a `std::time::Duration` value to represent the duration for which the character is slowed down.

The `Lightning` type _(4)_ does not include any value, as a character receiving a `lightning` attack dies immediately.

The `Poison` type _(5)_ includes an `i32` integer type for the total damage dealt and a `std::time::Duration value` to represent the duration over which the character receives damage.

With the `Damage` enum defined, we can now create instances of the `Damage` type. Instantiating an enum involves explicitly using the enum's name followed by one of its variants, as we have seen in the `assert_eq!` statement at the beginning of the post.

```rs
let physical = Damage::Physical {
    critical_hit: false,
};
let fire = Damage::Fire(12);
let cold = Damage::Cold(4, std::time::Duration::from_secs(3));
let lightning = Damage::Lightning;
let poison = Damage::Poison(2, std::time::Duration::from_secs(6));
```

For a simple retrieval of an enum value, we can use `if let` control flow operator.

```rust
let fire = Damage::Fire(12);

if let Damage::Fire(a) = fire {
  println!("Received {} fire damage", a)
}
```

Using the keyword `match`, we can use pattern matching to handle each variant.

```rust
fn take_damage(damage: Damage) {
    match damage {
        Damage::Physical { critical_hit } => {
            if critical_hit {
                println!("Received critical_hit damage. Character died")
            } else {
                println!("Received physical damage")
            }
        }
        Damage::Fire(a) => println!("Received {} fire damage", a),
        Damage::Cold(a, d) => {
            println!("Received {} cold damage. Slowed for {:?} seconds", a, d)
        }
        Damage::Lightning => println!("Received lightning damage. Character died"),
        Damage::Poison(a, d) => {
            println!("Received {} poison damage over {:?} seconds", a, d)
        }
    }
}
```

Another characteristic of enums in Rust is that they can implement methods.

```rust
enum Damage {
    // [...]
}

impl Damage {
    fn parity_damage(&self, n: i32) -> Damage {
        if n % 2 == 0 {
            Damage::Fire(12)
        } else {
            Damage::Lightning
        }
    }
}
```

[Enums](https://doc.rust-lang.org/book/ch06-00-enums.html) in Rust, as seen with the `Damage` enum example, are a powerful tool for grouping and organising related types. We can include any type of data or methods, making them extremely versatile.
