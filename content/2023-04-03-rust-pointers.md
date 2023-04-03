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
|[RefCell](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)|❌|Allows to mutate data even if there are immutable references to that data. Borrowing rules are enforced at runtime.|
|[Cow](https://doc.rust-lang.org/std/borrow/enum.Cow.html)|❌|Clone on write. Provides immutable access to borrowed data, and clone the data lazily when mutation or ownership is required.|
|[Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html)|✅|Atomically Reference Counted. Provides read shared ownership of data allocated in the heap. (~thread safe Box + Rc)|
|[Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html)|✅|Mutual exclusion to shared data. The data is only ever accessed when the mutex is locked and it blocks other thread attempting to access it. Beware of [poisoning](https://doc.rust-lang.org/std/sync/struct.Mutex.html#poisoning) that occurs if a thread holding the mutex panics.|
|[RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html)|✅|Reader-Writer lock. Allows at most 1 writer or any number of readers. Beware of [poisoning](https://doc.rust-lang.org/std/sync/struct.RwLock.html#poisoning) that occurs if a writer holding the mutex panics.|

Examples: [https://github.com/aurelien-clu/tutorial-rust/blob/main/crates/smart-pointers/src/main.rs](https://github.com/aurelien-clu/tutorial-rust/blob/main/crates/smart-pointers/src/main.rs)

## Box

```rust
// ❌ does not compile
// error[E0072]: recursive type Node has infinite size
struct Node {
    child: Option<Node>,
    value: usize,
}
let node = Node { value: 0, child: Some(Node{value: 1, child: None}) };

// ✅ compiles
struct Node {
    child: Option<Box<Node>>,
    value: usize,
}
let node = Node {
    value: 0,
    child: Some(Box::new(Node {
        value: 1,
        child: None,
    })),
};
```

Values stored on the stack are required to have their size known at compile time.

`Option<Node>` size is not known whereas `Option<Box<Node>>` is known, because `Box` holds a reference and its size is known.

The same goes with:

```rust
trait Foo {
    // [...]
}

// ❌ does not compile
// error[E0277]: the size for values of type `(dyn Foo + 'static)` cannot be known at compilation time
struct Bar {
    foo: dyn Foo,
}

// ✅ compiles
struct Bar {
    foo: Box<dyn Foo>,
}
```

## Rc - Reference Counted

Let's have multiple ownership over the same value `a`.

```rust
struct Foo {
    value: String,
}

let a = Rc::new(Foo { value: "a".to_string() });
println!("count after creating a = {}", Rc::strong_count(&a)); // 1

// adding a first reference
let b = Rc::clone(&a);
println!("count after creating b = {}", Rc::strong_count(&a)); // 2
{
    // adding a second reference inside the scope {}
    let c = Rc::clone(&a);
    println!("count after creating c = {}", Rc::strong_count(&a)); // 3
    // second reference goes out of scope
}
println!("count after c goes out of scope = {}", Rc::strong_count(&a)); // 2
```

## Refcell

⚠️ Borrowing rules are enforced at runtime => we are able to compile unsound code.

```rust
// ❌ compiles but... but fails at runtime
let x: RefCell<Vec<usize>> = RefCell::new(vec![]);
let mut mut_borrow_1 = x.borrow_mut();
let mut mut_borrow_2 = x.borrow_mut();
mut_borrow_1.push(1);
mut_borrow_2.push(2);
```

```rust
// ✅ compiles and works at runtime
let x: RefCell<Vec<usize>> = RefCell::new(vec![]);
{
    let mut mut_borrow_1 = x.borrow_mut();
    mut_borrow_1.push(1);
    // dropping mut_borrow_1 => we can mutably borrow x again
}
let mut mut_borrow_2 = x.borrow_mut();
mut_borrow_2.push(2);
```

## Arc - Atomically Reference Counted

```rust
// ❌ does not compile
// error[E0382]: use of moved value: `x`
// => x cannot be shared across threads
let x = Foo{_value: 0};
for _ in 0..3 {
    thread::spawn(move || {
        println!("{x:?}");
    });
}

// ✅ compiles
let x = Arc::new(Foo { _value: 5 });
for i in 0..3 {
    let x = Arc::clone(&x);

    thread::spawn(move || {
        println!("thread {i}, {x:?}");
    });
}
```

## Cow - Copy on write

Let's see when a copy on write occurs with the following examples:

```rust
fn abs_all(input: &mut Cow<[i32]>) {
    for i in 0..input.len() {
        let v = input[i];
        if v < 0 {
            // Clones into a vector if not already owned.
            input.to_mut()[i] = -v;
        }
    }
}

// No clone occurs because `input` doesn't need to be mutated.
let input = [0, 1, 2];
let mut input_as_cow = Cow::from(&input[..]);
abs_all(&mut input_as_cow);
assert_eq!(input.as_ptr(), input_as_cow.as_ptr()); // clone did not occur

// 📋 Clone occurs because `input` needs to be mutated.
let slice = [-1, 0, 1];
let mut input = Cow::from(&slice[..]);
abs_all(&mut input);
assert_ne!(input.as_ptr(), input_as_cow.as_ptr()); // clone occurred

// No clone occurs because `input` is already owned
let mut input = Cow::from(vec![-1, 0, 1]);
let initial_address = format!("{:p}", input.as_ptr());
abs_all(&mut input);
let address_afterward = format!("{:p}", input.as_ptr());
assert_eq!(initial_address, address_afterward); // clone did not occur
```
