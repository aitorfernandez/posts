# Dungeons, Trees and Grids

I've been playing around trying to generate a map for a kind of Zelda-Diablo game mechanics.

One of the main characteristics of Diablo is the randomization of game maps, and one of the characteristics of Zelda is the puzzles, particularly in dungeons.

In this post, I'll introduce an abstraction of the algorithm I've created to mix the two main characteristics, randomization and dungeon puzzles.

## The problem

Thinking of a very basic dungeon, I would say that every dungeon has some *special* rooms and a random number of *normal-combat* rooms. For example, a possible representation of a six rooms dungeon can be:

```txt
+---+---+---+
| 3 | 1 ~ 2 |
+---+-~-+---+
| 1 ~ 0 ~ 1 |
+---+---+---+

0. Entrance room / 1. Normal-Combat rooms / 2. Key room / 3. Boss room
```

The player starts in room 0, and the goal is to defeat the boss in room 3. However, the player can't get into the boss room without unlocking the door and the player can't open the door without obtaining the key in room 2.

Although the dungeon looks relatively simple, with the only puzzle being to find the key, if the dungeon is generated randomly, there is a possibility that the key may appear after the boss room. In this case, the player cannot access the key room, and the dungeon will never be completed.

## The algorithm and the Floor builder

To resolve the different steps in the algorithm, I've used a builder pattern to handle each step.

```rust
pub struct Floor {
    // [...]
}

impl Floor {
    pub fn builder() -> FloorBuilder {
        FloorBuilder::default()
    }
}

#[derive(Default)]
pub struct FloorBuilder {
    // [...]
}

impl FloorBuilder {
    // [...]
    rooms: Vec<Room>,
}
```

The `impl FloorBuilder` will have one method for each step.

### 1st step: The Linear Structure

In the first step, the algorithm creates the total rooms in order, in a linear structure, all elements except the first and the last, have predecessor and successor.

```rust
impl FloorBuilder {
    #[must_use]
    pub fn total_rooms(self, total_rooms: u8) -> Self {
        Self {
            total_rooms,
            ..self
        }
    }

    #[must_use]
    pub fn add_rooms(self) -> Self {
        let rooms: Vec<_> = (0..self.total_rooms).map(Room::new).collect();

        Self { rooms, ..self }
    }
}
```

After the builder calls the methods `total_rooms` and `add_rooms` the `Vec<Room>` end up with something like

```txt
+---+---+---+---+---+---+
| 1 | 1 | 1 | 1 | 1 | 1 |
+---+---+---+---+---+---+
```

The `Room::new` creates a new room with the default room type.

```rust
pub enum RoomType {
    Entrance,
    Combat,
    Key,
    Boss,
}

impl Default for RoomType {
    fn default() -> Self {
        Self::Combat
    }
}
```

### 2nd step: The Theme

The next step in the `FloorBuilder` is the `add_theme` method. With a linear data structure where each room is one element in the array, I can be 100% sure that the first element will be the start room, and the last element (or the last element - n) in the array will be the boss room. Any random element between the first and the boss room can be the key room.

```rust
impl FloorBuilder {
    // [...]

    #[must_use]
    pub fn add_theme(mut self) -> Self {
        let mut rng = thread_rng();

        self.rooms[0].room_type = RoomType::Entrance;
        self.rooms[(self.total_rooms - 1) as usize].room_type = RoomType::Boss;

        let n = rng.gen_range(2..self.total_rooms - 1);
        self.rooms[n as usize].room_type = RoomType::Key;

        self
    }
}
```

And the `Vec<Room>` now looks like this:

```txt
+---+---+---+---+---+---+
| 0 | 1 | 1 | 1 | 2 | 3 |
+---+---+---+---+---+---+
```

### 3rd step: The Tree

The third step in the algorithm is to convert the array into a [Level Order Tree](https://en.wikipedia.org/wiki/Breadth-first_search).

The Level Order Tree obtained ensures that the dungeon is properly connected and that the player can progress from the entrance room to the boss room. The tree structure allows for efficient traversal of the dungeon, as each room is connected to its neighbours through its children in the tree.

```rust
impl FloorBuilder {
    // [...]

    #[must_use]
    pub fn bfs_tree(mut self) -> Self {
        let mut rng = thread_rng();

        let mut ids: Vec<_> = (0..self.total_rooms).collect();

        ids.reverse();

        let mut queue = VecDeque::new();
        queue.push_back(ids.pop().unwrap());

        let mut total_children = 0;

        while let Some(id) = queue.pop_front() {
            total_children = if id == 0 {
                rng.gen_range(1..=4)
            } else if total_children == 0 {
                rng.gen_range(1..=2)
            } else {
                rng.gen_range(0..=2)
            };

            if total_children == 0 && self.rooms[(id - 1) as usize].nodes.len() == 1 {
                total_children = rng.gen_range(1..=2);
            }

            for _ in 0..total_children {
                if let Some(i) = ids.pop() {
                    self.rooms[id as usize].add_node(i);
                    queue.push_back(i);
                }
            }
        }

        self
    }
}
```

In this step, the algorithm iterates over each element in the array and assigns a random number of children to it, ranging from 1 to 2 (or from 1 to 4 if the element is the entrance room).

After this step, the array from the previous step is updated, and the expected output of nodes should look like this:

```txt
   1
 / | \
2  3  4
   |\
   5 6
```

### 4th step: The Grid

In this step, the algorithm adds a coordinate to each room in the `Vec<Room>` using the nodes created in the previous step. For each room, the algorithm checks if it has children. If it does, a random direction (North, South, East, or West) is chosen from the free coordinates around the parent room, and the child is placed there.

```rust
impl FloorBuilder {
    // [...]

    #[must_use]
    pub fn grid_placement(mut self) -> Self {
        let mut occupied_locs = HashSet::new();
        occupied_locs.insert(self.rooms[0].coord.clone());

        for i in 0..self.rooms.len() {
            for j in 0..self.rooms[i].nodes.len() {
                let id = self.rooms[i].nodes[j] as usize;

                let coord = self.rooms[i].coord.rand_coord_around(&occupied_locs);
                self.rooms[id].coord = self.rooms[i].coord.plus(&coord);
                occupied_locs.insert(self.rooms[id].coord.clone());

                let direction = coord.into();
                // parent
                self.rooms[i].add_connected_room((id as u8, direction));
            }
        }

        self
    }
}
```

After this step, with each room/node having a coordinate the dungeon map might look something like this:

```txt
+---+---+---+
| 6 | 3 | 5 |
+---+---+---+
| 2 | 1 | 4 |
+---+---+---+
```

The solution is pretty cool, and I have a new random dungeon each time. All key rooms are in order and I can use the id/node in each room to increase the difficulty as the player progresses.
