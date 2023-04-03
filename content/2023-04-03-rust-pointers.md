+++
title = "Rust smart pointers"
weight = 1
order = 1
date = 2023-04-03
insert_anchor_links = "right"
[taxonomies]
tags = ["rust", "pointers", "tldr"]
+++

## Common Rust smart pointers

|name|thread-safe|description|
|----|-----------|-----------|
|[Box](https://doc.rust-lang.org/book/ch15-01-box.html)|❌|Boxes allow you to store data on the heap rather than the stack. Useful for unknown size at compile time (e.g. recursive objects, lists, dynamic types), prevents data copy.|
|[Rc](https://doc.rust-lang.org/book/ch15-04-rc.html)|❌|Reference Counted: enable multiple ownership to the same value.|
|[RefCell](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)|❌|Allows to mutate data even if there immutable references to that data. Borrowing rules are enforced at runtime.|
|[Cow](https://doc.rust-lang.org/std/borrow/enum.Cow.html)|❌|Clone on write. Provides immutable access to borrowed data, and clone the data lazily when mutation or ownership is required.|
|[Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html)|✅|Atomically Reference Counted. Provides read shared ownership of data allocated in the heap. (~thread safe Box + Rc)|
|[Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html)|✅|Mutual exclusion to shared data. The data is only ever accessed when the mutex is locked and it blocks other thread attempting to access it. Beware of [poisoning](https://doc.rust-lang.org/std/sync/struct.Mutex.html#poisoning) that occurs if a thread holding the mutex panics.|
|[RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html)|✅|Reader-Writer lock. Allows at most 1 writer or any number of readers. Beware of [poisoning](https://doc.rust-lang.org/std/sync/struct.RwLock.html#poisoning) that occurs if a writer holding the mutex panics.|

Examples: [https://github.com/aurelien-clu/tutorial-rust/blob/main/crates/smart-pointers/src/main.rs](https://github.com/aurelien-clu/tutorial-rust/blob/main/crates/smart-pointers/src/main.rs)
