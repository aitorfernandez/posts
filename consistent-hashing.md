# Consistent Hashing

One way to balance load in a system is by using consistent hashing. Consistent hashing lets us map a hashed request to a "ring" of servers. With this approach, we can easily add or remove servers based on traffic or other circustances, and if one server goes down or gets too busy, the request will simply be directed to the next available server in the ring.

From this introduction, we can pull out some key ideas:

1. Servers: These handle the requests. We can have multiple virtual servers, where each virtual server represents a position on the ring.

2. The ring (or hash space): This is, in my opinion, the most interesting concept. Every element, whether it's a server or a request, gets mapped to a position on the ring.

3. Hashing: As mentioned earlier, all elements are positioned on the ring based on their hash values.

4. Assignment: This is the logic that decides which available server in the ring should handle the request.

Even though consistent hashing can be tricky to implement, I’ll try writing some code to better understand the ideas behind it and see a simple version of this concept in action. Let’s roll!

## Servers

Without servers, we can't handle requests, so first thing first, we'll implement the Server struct, which we can later add to the ring.

```rust
#[derive(Debug)]
struct Server {
    id: u32,
}

impl Server {
    fn new(id: u32) -> Self {
        Self { id }
    }
}
```

That’s it, a simple server. Now we can create a Vec of servers to move around and work with them.

```rust
fn main() {
    let servers = vec![
        Arc::new(Server::new(1)),
        Arc::new(Server::new(2)),
        Arc::new(Server::new(3)),
    ];
}
```

Our servers are ready, but having servers always up is hard to believe. In a real scenario, servers can go down for various reasons. To simulate these downtimes, we’re going to add some randomness to the server implementation.

We'll introduce a couple of dependencies: `tokio` for async and threading operations, and `rand` for generating randomness.

```toml
[package]
# [...]

[dependencies]
tokio = { version = "1", features = ["full"] }
rand = { version = "0.8", features = ["std_rng"] }
```

Now, let's add a flag field to track whether a server is active or down.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug)]
struct Server {
    // [..]
    active: Arc<Mutex<bool>>,
}

impl Server {
    fn new(id: u32) -> Self {
        let active = Arc::new(Mutex::new(true));
        Self { id, active }
    }

    async fn is_active(&self) -> bool {
        *self.active.lock().await
    }
}
```

Let's pause here for a moment. What does `Arc<Mutex<bool>>` mean? Later on, we'll have virtual servers running in the ring, and the server will be shared across different asynchronous tasks. `Arc` allows the active field to be cloned and shared safely across those tasks. `Mutex` gives us a way to lock the active field when a task needs to access or modify it. This ensures that we can update the active field safely to simulate a server being up or down.

```rust
// [..]
use tokio::{sync::Mutex, time::sleep};

// [..]

impl Server {
    // [..]

    async fn take_down(&self, secs: u64) {
        {
            let mut active = self.active.lock().await;
            *active = false;
            println!("Server {} is down", self.id);
        }

        sleep(Duration::from_secs(secs)).await;

        {
            let mut active = self.active.lock().await;
            *active = true;
            println!("Server {} is back up", self.id);
        }
    }
}
```

To finish up the server setup, we'll create a helper function that randomly shuts down our servers. Inside this helper, an infinite loop will shut down the servers one at a time, for simplicity, with 4 seconds between each shutdown.

```rust
use rand::{rngs::StdRng, Rng, SeedableRng};
// [...]

// [...]

async fn take_down_servers(servers: Vec<Arc<Server>>) {
    let mut rnd = StdRng::from_entropy();

    loop {
        let idx = rnd.gen_range(0..servers.len());
        let server = servers[idx].clone();

        tokio::spawn(async move { server.take_down(4).await });

        sleep(Duration::from_secs(4 - 1)).await;
    }
}

#[tokio::main]
async fn main() {
    // [...]

    tokio::spawn(take_down_servers(servers.clone()));
}
```

## Hashing and Ring Concept

The servers are virtually positioned in the ring. To represent the ring, we’re going to use a `BTreeMap` data structure. A `BTreeMap` organizes elements as key-value pairs and keeps the keys sorted automatically. `BTreeMap`, it’s like magic: fast sorting and searching all in one!

```rust
// [...]
use std::{
    collections::BTreeMap,
    sync::Arc,
};

struct ConsistentHashing {
    ring: BTreeMap<u64, Arc<Server>>,
}

impl ConsistentHashing {
    fn new(servers: Vec<Arc<Server>>) -> Self {
        let mut ring = BTreeMap::new();

        for server in servers {
            for i in 0..3 {
                let key = calculate_hash(format!("{i}{}", server.id));
                ring.insert(key, server.clone());
            }
        }

        Self { ring }
    }
}
```

The `calculate_hash` helper function takes an input and calculates the hash, the unique "fingerprint" for the server. We delegate this task to `DefaultHasher`, a built-in tool in Rust that knows how to generate a hash value.

```rust
// [...]
use std::{
    // [...]
    hash::{DefaultHasher, Hash, Hasher},
};

// [...]

fn calculate_hash<T: Hash>(t: T) -> u64 {
    let mut s = DefaultHasher::new();
    t.hash(&mut s);
    s.finish()
}
```

The `ConsistentHashing` constructor will clone 3 copies of each server, with each copy assigned to a hash key in the ring for quick lookup and to easily find the server.

```rust
#[tokio::main]
async fn main() {
    // [...]

    let consistent_hashing = Arc::new(ConsistentHashing::new(servers.clone()));
}
```

By calling the constructor, we can imagine the result as a ring, where the keys are evenly spaced around the circle based on their values, ensuring load balancing.

```
                        (server 1-0)
                            18352...
        (server 1-1)                   (server 2-0)
            14873...                   18441...

(server 2-2)                                    (server 3-0)
    10843...                                    36001...

        (server 3-1)                   (server 1-2)
            68568...                   15055...
                        (server 3-2)
                            22285...
```

## Assignment

For the last part, request assignment, we need to assign each incoming request to a server in the ring. When we receive a request, we create a new hash. This hash for sure will not be exactly the same as the ones in our ring, but it will be close to one of the server hashes in the ring.

Let’s create a new function to get the server for an incoming request.

```rust
// [...]

impl ConsistentHashing {
    // [...]

    async fn get_server_for_request(&self, request: &str) -> Option<Arc<Server>> {
        let hash_value = calculate_hash(request);
        let servers = self.ring.range(hash_value..).chain(self.ring.range(..hash_value));

        for (_, server) in servers {
            if server.is_active().await {
                return Some(server.clone());
            }
        }

        None
    }
}
```

The `hash_value` variable is the new value, which tells us where to start looking for servers. Since we're working with a ring, we use "chain" to combine the ranges and avoid issues when the hash falls near the end of the ring.

For example, if the servers are positioned at [100, 300, 500, 800, 1000] and the request hash value is 600, the first part of the search will cover [800, 1000], and we chain it to [100, 300, 500]. This way, we ensure that all servers are checked in the correct clockwise order around the ring.

Finally, we can create the fake request in the main function and test it:

```rust
#[tokio::main]
async fn main() {
    // [...]

    let mut requests = vec![];
    for i in 0..100 {
        requests.push(format!("Request {i}"));
    }

    for request in requests {
        let consistent_hashing_clone = Arc::clone(&consistent_hashing);

        tokio::spawn(async move {
            match consistent_hashing_clone
                .get_server_for_request(&request)
                .await
            {
                Some(server) => println!("{request} handle by server {}", server.id),
                None => panic!("No server available for {request}"),
            }
        });

        sleep(Duration::from_secs(1)).await;
    }
}
```

And the result could look something like this:

```
Server 3 is down
Request 0 handle by server 1
Request 1 handle by server 2
Request 2 handle by server 2
Server 1 is down
Request 3 handle by server 2
Server 3 is back up
Request 4 handle by server 2
Request 5 handle by server 3
```

If a server is down, other servers will handle the request. Nice!
