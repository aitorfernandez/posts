# Sharding With Token Coordination

Trying to mimic real-world scenarios is always a good exercise, even though we know that real-world situations are usually more complex. However, these exercises are a great starting point for understanding the problem.

In this post, we'll be focusing on two key concepts: sharding and token-based coordination.

The main idea behind sharding is to divide data into smaller and more manageable parts called "shards". Each shard can then be handled independently, which helps improve scalability and parallelism.

The concept of a token can be a bit abstract, and it can mean different things depending on the domain. In this example, we'll use the token approach to help coordinate and maintain the order of our shards. This ensures, for example, that no shard ends up taking more time to process than the others.

## Sharding

For the sharding part, we're going to work with some `Readers`. These `Readers` can be thought of as nodes in a distributed system, where each one represents a portion of a larger dataset. Now, let's write some simple code to demonstrate this:

```rust
struct Reader {
    id: u32,
}

impl Reader {
    fn new(id: u32) -> Self {
        Self { id }
    }
}
```

That was easy! It's always a good idea to start small. Now, let's implement the `Reader` to handle small portions of data, like reading a CSV file, for example.

```toml
# [...]

[dependencies]
csv = "1.3"
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Data {
    #[serde(rename(deserialize = "value"))]
    val: u32,
}

// [...]

impl Reader {
    // [...]
    fn read(&self) {
        let file = format!("data/{}.csv", self.id);
        let mut rdr = csv::ReaderBuilder::new()
            .has_headers(true)
            .trim(csv::Trim::All)
            .from_path(file)
            .expect("Error: Failed to read from file");

        let mut data = rdr.deserialize::<Data>();

        while let Some(result) = data.next() {
            match result {
                Ok(Data { val }) => {
                    println!("Reader {} emmit value {val}", self.id);
                }
                Err(e) => {
                    eprintln!("Error: Failed to deserialize {e}");
                    continue;
                }
            }
        }
    }
}
```

Using the id field from each `Reader`, we'll load and read a CSV file. The next step will be to run multiple Readers to simulate a distributed system. We'll be running each `Reader` in separate concurrent tasks using the Tokio crate.

```toml
# [...]

[dependencies]
# [...]
tokio = { version = "1.42", features = ["full"] }
```

```rust
// [...]
impl Reader {
    // [...]
    async fn read(&self) {
        // [...]
    }
}

#[tokio::main]
async fn main() {
    let readers: Vec<Reader> = (0..=4).map(|i| Reader::new(i)).collect();

    let mut handles = vec![];

    for reader in readers {
        let handle = tokio::spawn(async move {
            reader.read().await;
        });

        handles.push(handle);
    }

    // Wait for all tasks to finish...
    for h in handles {
        let _ = h.await;
    }
}
```

Using `tokio::spawn` creates a new task for each `Reader`, and all these tasks will be scheduled and run by the Tokio runtime. However, there's no guarantee about the order in which they'll execute. Running the application we could have something like this:

```shell
Reader 3 emmit 30
Reader 2 emmit 20
Reader 0 emmit 0
Reader 1 emmit 10
Reader 4 emmit 40
Reader 1 emmit 11
Reader 0 emmit 1
...
```

Although there's no guarantee about the order, we managed to simulate file sharding in a really basic way, where each `Reader` handles a specific chunk of data.

To continue mimicking real-world concepts, the next step is to centralize all the information received from the Readers into one point.

Since the Readers are running in separate tasks, we need a way for them to communicate. For this, we'll use Tokio channels. In simple terms, channels allow different parts of the running application to send messages to each other.

The first step is to define the type of message we want to send.

```rust
// [...]
use std::fmt;

// [...]

struct Message {
    id: u32,
    val: u32,
}

impl fmt::Display for Message {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Reader {} emmit value {}", self.id, self.val)
    }
}
```

Next, we'll create the centralized data collector.

```rust
use tokio::{sync::mpsc};

struct DataCollector {
    receiver: mpsc::Receiver<Message>,
}

impl DataCollector {
    fn new(receiver: mpsc::Receiver<Message>) -> Self {
        Self { receiver }
    }

    async fn collect(&mut self) {
        while let Some(message) = self.receiver.recv().await {
            println!("{message}");
        }
    }
}
```

The `receiver` field in the DataCollector struct is of type `mpsc::Receiver`, which will be receiving messages from multiple producers. This might seem a bit abstract, but if you keep this [Futurama clip](https://www.youtube.com/watch?v=6hfcnm8V4fA&t=35s) in mind, it'll make a lot more sense—and we'll never forget it again the concept of a channel.

The `while` loop in the `collect` function keeps receiving and displaying messages one at a time until there are no more messages to receive.

With the receiver set up, the next step is to prepare the senders:

```rust
#[tokio::main]
async fn main() {
    // [...]

    let (message_tx, message_rx) = mpsc::channel(96);

    let mut data_collector = DataCollector::new(message_rx);
    tokio::spawn(async move {
        data_collector.collect().await;
    });

    let mut handles = vec![];

    for reader in readers {
        let message_sender = message_tx.clone();

        let handle = tokio::spawn(async move {
            reader.read(message_sender).await;
        });

        // [...]
    }

    // [...]
}
```

Now, let's use the senders in the `read` function.

```rust
use std::{fmt, time::Duration};
use tokio::{
    sync::mpsc,
    time::sleep,
};

// [...]

impl Reader {
    // [...]

    async fn read(&self, message_sender: mpsc::Sender<Message>) {
        // [...]

        while let Some(result) = data.next() {
            match result {
                Ok(Data { val }) => {
                    if let Err(e) = message_sender.send(Message { id: self.id, val }).await {
                        eprintln!("Error: {e}");
                    }
                }
                Err(e) => {
                    // [...]
                }
            }
            // Simulating som processing time
            sleep(Duration::from_secs(1)).await;
        }
    }
}

// [...]
```

By removing the `println!` and adding the `message_sender`, we can send the message, which will then be received by the receiver in the `DataCollector`.

When we run the application, the result will be similar to the previous output, but now the information is centralized. This makes it much easier to extend or validate the data if needed, since everything is gathered in one place. Yay!

```shell
Reader 0 emmit value 0
Reader 2 emmit value 20
Reader 1 emmit value 10
Reader 3 emmit value 30
Reader 4 emmit value 40
Reader 1 emmit value 11
Reader 4 emmit value 41
Reader 3 emmit value 31
...
```

## Token Coordination

The final step is to use a token-based approach to bring some order to the data. The idea behind token coordination is to give each `Reader` an equal opportunity to send data.

For the token coordination, we'll use a different type of channel. While for the Readers we used an `mpsc` (multiple producers, single consumer) channel, for the token we'll use a `watch` channel, which allows multiple producers and multiple consumers.

There will only be one producer, the token coordinator, but multiple consumers, the Readers. I think `watch` fits well with the idea of shared files, as Readers can observe the token and act independently when it's their turn.

Now, let's start implementing the TokenController.

```rust
// [...]
use tokio::sync::{mpsc, watch};

// [...]

struct TokenController {
    total_readers: u32,
    sender: watch::Sender<u32>,
}

impl TokenController {
    fn new(total_readers: u32, sender: watch::Sender<u32>) -> Self {
        Self {
            total_readers,
            sender,
        }
    }

    async fn run(&self) {
        let mut current_token = 0;
        loop {
            self.sender.send(current_token).unwrap();

            sleep(Duration::from_secs(3)).await;

            current_token = (current_token + 1) % self.total_readers;
        }
    }
}

// [...]
```

The `TokenController` in the `run` function sends the tokens in a round-robin fashion way, and the cycle repeats infinitely. The token is moved every 3 seconds, giving all readers the same amount of time to process their data.

This is pretty cool, just like we centralized the data, here we can update the run function to support a round-robin approach with a weighted system, all without affecting other parts of the application.

With the `TokenController` ready to send the tokens, the next step is to pass the token receiver to each reader.

```rust
#[tokio::main]
async fn main() {
    // Even the id and the token are the same, we pass it as separate values, for separate concepts
    let readers: Vec<Reader> = (0..=4).map(|i| Reader::new(i, i)).collect();

    // [...]

    let (token_tx, token_rx) = watch::channel(0);

    // [...]

    let token_controller = TokenController::new(readers.len() as u32, token_tx);
    tokio::spawn(async move {
        token_controller.run().await;
    });

    // [...]

    for reader in readers {
        let message_sender = message_tx.clone();
        let token_receiver = token_rx.clone();

        let handle = tokio::spawn(async move {
            reader.read(token_receiver, message_sender).await;
        });

        // [...]
    }

    // [...]
}
```

Now, let's update the logic in the reader.

```rust
// [...]

struct Reader {
    id: u32,
    token: u32,
}

impl Reader {
    fn new(id: u32, token: u32) -> Self {
        Self { id, token }
    }

    async fn read(
        &self,
        token_receiver: watch::Receiver<u32>,
        message_sender: mpsc::Sender<Message>,
    ) {
        // [...]

        let mut token_receiver = token_receiver.clone();

        while let Some(result) = data.next() {
            while *token_receiver.borrow() != self.token {
                // waits until the TokenController sends a new value to the watch channel
                token_receiver.changed().await.unwrap();
            }

            // [...]
        }
    }
}
```

What did we do? The added `while` loop ensures that the Reader only proceeds when the token received matches its own token. If it doesn't match, the Reader waits for the `TokenController` to send a new value to the watch channel.

When we run the application, we should see things running in a much more controlled way.

```shell
Reader 0 emmit value 0
Reader 0 emmit value 1
Reader 0 emmit value 2
Reader 1 emmit value 10
Reader 1 emmit value 11
Reader 1 emmit value 12
Reader 2 emmit value 20
Reader 2 emmit value 21
Reader 2 emmit value 22
Reader 3 emmit value 30
Reader 3 emmit value 31
Reader 3 emmit value 32
Reader 4 emmit value 40
Reader 4 emmit value 41
Reader 4 emmit value 42
Reader 0 emmit value 3
Reader 0 emmit value 4
Reader 0 emmit value 5
...
```

And that's it! While this really simple to be a real-world implementation, it's a clean and scalable approach that can be adapted to handle larger, more complex systems.
