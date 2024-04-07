# Type System for Correctness

Recently, in a pet project, I started to create a server with the idea of use the same server across all microservices within the application. However, instantiating the server, I often find myself forgetting certain values or not creating them in the correct order.

I started to think how Rust's [Result](https://doc.rust-lang.org/std/result/enum.Result.html) enum is a perfect example of using the type system for correctness for error handling. So, to ensure correctness while developing the server in my pet project, I should employ a similar approach.

For example, the server requires a name, among other values, for its creation. It look something like this:

```rust
Servify::builder()
    .with_name("hello-world-server")
    // [...]
    .build()
    .await
```

At the moment of building the server in the `build` method, it requieres all fields such as name, service, timeouts, etc. How can we ensure that we have all the necessary fields? Before focusing on the fields, let's examine the three stages involved in the process of creating the server using a builder pattern.

```rust
// Stage 1: The builder provides methods for customization.
Servify::builder()
    // Stage 2: The customization methods
    .with_name("hello-world-server")
    // [...]
    // Stage 3: The construction of the server
    .build()
    .await
```

For stage 1, we prepare the builder with all fields unassigned. In stage 2, although fields are not yet assigned, we start the process of assigning them. Finally, we can progress to stage 3 only if all fields have been assigned.

Reading the previous text, we can identify a pattern wherein the fields transition from being **unassigned** to being **assigned**. We can easily prepare a module to handle the states of the fields.

```rs
mod field_state {
    pub struct Unassigned;
    pub struct Assigned<T>(pub T);
}
```

Now, we can prepare the first stage and set all fields to the state of `Unassigned`. We'll utilize a [Default](https://doc.rust-lang.org/std/default/trait.Default.html) trait for the initialization of the builder.

```rust
pub struct Servify;

impl Servify {
    #[must_use]
    pub fn builder() -> ServifyBuilder<field_state::Unassigned /* [...] */> {
        ServifyBuilder::default()
    }
}

pub struct ServifyBuilder<N> {
    pub(crate) name: N,
    // [...]
}

impl Default for ServifyBuilder<field_state::Unassigned /* [...] */> {
    fn default() -> ServifyBuilder<field_state::Unassigned /* [...] */> {
        ServifyBuilder {
            name: field_state::Unassigned,
            // [...]
        }
    }
}
```

The second stage is interesting, particularly the return in the function signature, where we return the builder with the field **assigned** with the correct type.

```rust
impl<N> ServifyBuilder<N> {
    pub fn with_name(
        &self,
        name: &'static str,
    ) -> ServifyBuilder<field_state::Assigned<&'static str>> {
        ServifyBuilder {
            name: field_state::Assigned(name),
        }
    }

    // [...]
}
```

And finally, in the third stage, we ensure that the build method receives the correct values by creating a new implementation for the `ServifyBuilder`. This time, we specify in the implementation that we require `Assigned` values.

```rs
impl ServifyBuilder<field_state::Assigned<&'static str> /* [...] */> {
    pub async fn build(self) -> anyhow::Result<()> {
        // [...]
    }
}
```

Now, if for example we fail to pass the name to the builder.

```rust
Servify::builder().build().await
```

The compiler notifies us with a message such as **no method build for** `ServifyBuilder<Unassigned>`, preventing us from moving to the third stage if we haven't previously assigned all the necessary fields. We are using the types inside `field_state` module to ensure correctness ❤️
