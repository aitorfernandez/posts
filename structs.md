# Structs

In the [Enum](https://aitorfernandez.hashnode.dev/rust-enums) post, we saw how an [Enum](https://aitorfernandez.hashnode.dev/rust-enums) allows us to have one specific value at a time. On the other hand, Rust structs provide a way to hold multiple related values. Structs let us bundle together related properties into a single type.

There are three types of structs, `named-field`, `tuple-like`, and `unit-like`. We'll leave `unit-like` structs for another post about Traits, and in this post, we'll mainly focus on `named-field` and `tuple-like` structs. The difference between these two structs is how we access their values.

`Tuple-like` structs identify their values by order. For example, we can represent a `Color` type as a `tuple-like`.

```rs
struct Color(u8, u8, u8)

let c = Color(255, 165, 0);

assert_eq!(c.0, 255); // R
assert_eq!(c.1, 165); // G
assert_eq!(c.2, 0);   // B
```

Another example of using a `tuple-like` for representing X an Y coordinates can be:

```rs
struct Position(u32, u32)

let p = Position(12, 24);

assert_eq!(p.0, 12); // X
assert_eq!(p.1, 24); // Y
```

In contrast to `tuple-like` structs, `named-field` structs don't rely on order to identify their values. Instead, we assign a name to each element. The syntax for a `named-field` struct is similar to `tuple-like` struct, with the addition of field names and values, all enclosed within curly braces.

```rs
struct Position {
    x: u32,
    y: u32,
}

let p = Position { x: 12, y: 24 };

assert_eq!(p.x, 12); // X
assert_eq!(p.y, 24); // Y
```

A struct can contain other structures. Utilizing the above examples `Color` and `Position`, we can create a new struct that groups all elements into a single entity, such as a `Square` type struct.

```rs
struct Square {
    color: Color,
    position: Position,
    size: u32,
}
```

We have created a `Square` struct, which demonstrates that a struct is a combination of related elements.

If the `struct` keyword is used to define the type and elements of a struct, the implementation of the struct is accomplished using the `impl` keyword. With the implementation, we bring all elements together to make things more interesting.

Continuing with the `Square` struct, we can start implementing the square by adding a `new` method that returns a `Square`.

```rs
struct Square {
    // [...]
}

impl Square {
    fn new() -> Self {
        Self {
            color: Color(0, 0, 0),
            position: Position(0, 0),
            size: 200,
        }
    }
}
```

It's important to note the usage of `pub` and `Self`. By using `pub`, we make the `new` function public and we can create squares using `Square::new()`.

The `Self` keyword is a shorthand for the name of the struct. In this code, it serves as a replacement for `Square` type. The next code is equivalent.

```rs
// [...]

impl Square {
    pub fn new() -> Square {
        Square {
            color: Color(0, 0, 0),
            position: Position(0, 0),
            size: 100,
        }
    }
}
```

The current `Square` implementation and `new` method is not very interesting because all squares using the `new` method has the same color, position, and size. To make it more interesting, the [Rand](https://crates.io/crates/rand) crate can be utilized to create random color and position values.

```sh
cargo add rand
```

With the next code, we do the new implementation for returning random colors in the `Color` struct:

```rs
use rand::Rng;

struct Color(u8, u8, u8);

impl Color {
    pub fn new() -> Self {
        let mut rng = rand::thread_rng();
        Color(
            rng.gen_range(0..=u8::MAX),
            rng.gen_range(0..=u8::MAX),
            rng.gen_range(0..=u8::MAX),
        )
    }
}
```

The `gen_range` method from the Rand crate generates random u8 values between 0 and 255.

Similarly, we can do the same for the `Position` struct and return random values between 0 and 1024 for the x position and between 0 and 768 for the y position.

```rs
struct Position(u32, u32);

impl Position {
    pub fn new() -> Self {
        let mut rng = rand::thread_rng();
        Position(rng.gen_range(0..=1024), rng.gen_range(0..=768))
    }
}
```

We can update the `Square::new()` method to use the `new` `Position` method and `new` `Color` method.

```rs

struct Square {
    // [...]
}

impl Square {
    pub fn new() -> Self {
        let mut rng = rand::thread_rng();
        Self {
            color: Color::new(),
            position: Position::new(),
            size: rng.gen_range(12..=96),
        }
    }
}
```

Now, every time we call `Square::new()`, we'll get a square with different colors, positions, and sizes.

We can keep going with `named-field` structs style and create a new simple struct called `SquareFactory` to manage hundreds of squares. It'll look like this:

```rs
struct SquareFactory {
    squares: Vec<Square>,
}
```

We further enhance our `SquareFactory` struct by implementing a method that allows us to create a specified number of squares. This method will take an argument `n` that represents the number of squares desired and returns a new `SquareFactory` instance with the squares property populated with `n` `Square` instances.

```rs
// [...]

impl SquareFactory {
    pub fn new(n: u32) -> Self {
        let squares: Vec<Square> = (0u32..n).map(|_| Square::new()).collect();
        Self { squares }
    }
}
```

We first create a vector of `Square` instances by mapping over the range of `0u32` to `n` and creating a new `Square` instance for each iteration. Then collect into the squares variable and return the `SquareFactory` instance.

We return the `SquareFactory` using the shorthand `Self { squares }` instead `Self { squares: squares }`

Creating an instance of the `SquareFactory` with `n` squares, we call `SquareFactory::new(n)`:

```rs
// [...]

let squareFactory = SquareFactory::new(492);
```

We've done some great work with the `SquareFactory` and `Square` structs, it's time to see our colorful squares in action.

The [Image](https://crates.io/crates/image) crate offers a range of functions and methods to work with images. To add the Image crate to the project, we run:

```sh
cargo add image
```

Using the Image crate, we can create a new `ImageBuffer` of a specified width and height and modify the pixels according to our squares instances. We can iterate over the image using a `for` loop and finally save the image as a .png file named squares.png.

```rs
// [...]

let mut img = image::ImageBuffer::new(1027, 768);

for (x, y, pixel) in img.enumerate_pixels_mut() {
    // ...
}

img.save("squares.png").unwrap();
```

With the image and loop set up, we can start preparing the code to run inside the `for` loop.

The first line inside the `for` loop sets each pixel to be dark blue.

```rs
// [...]

for (x, y, pixel) in imgbuf.enumerate_pixels_mut() {
    *pixel = image::Rgb([15, 23, 42]);
}

// [...]
```

Then, we'll go through each square in `squareFactory.squares` and check if the current pixel is in the range of the square. If it is, we can paint it with the square's color.

```rs
// [...]

for (x, y, pixel) in imgbuf.enumerate_pixels_mut() {
    *pixel = image::Rgb([15, 23, 42]);

    for square in squareFactory.squares.iter() {
        if square.in_range(x, y) {
            *pixel = image::Rgb(square.rgb());
        }
    }
}

// [...]
```

Let's create the `in_range` method and the `rgb` methods in the `Square` struct.

```rs
// [...]

impl Square {
    // [...]

    pub fn in_range(&self, x: u32, y: u32) -> bool {
        (x > self.position.0 && x < self.position.0 + self.size)
            && (y > self.position.1 && y < self.position.1 + self.size)
    }
}
```

This method will take two arguments, x and y positions from the image, and return a `bool` value indicating whether the given x and y are within the range of the square's size.

The `&self` in the method definition refers to a reference to the instance of the struct Square that the method is being called on. This allows the method to access and manipulate the properties of the struct without taking ownership of it.

Now we implement the `rgb` method.

```rs
impl Square {
    // [...]

    pub fn rgb(&self) -> [u8; 3] {
        [self.color.0, self.color.1, self.color.2]
    }
}
```

The `rgb` method returns an array with the random red, green and blue values of the square's color. This method is used to paint the pixels with the corresponding color of each square using `square.rgb()`

```rs
// [...]

for (x, y, pixel) in imgbuf.enumerate_pixels_mut() {
    // [...]

    for square in squareFactory.squares.iter() {
        if square.in_range(x, y) {
            *pixel = image::Rgb(square.rgb());
        }
    }
}

// [...]
```

If everything has been implemented correctly, the output should resemble a visual representation of multiple squares with randomly generated colors and positions, something like: