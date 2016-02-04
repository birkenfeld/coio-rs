# Coroutine I/O

[![Build Status](https://travis-ci.org/zonyitoo/coio-rs.svg?branch=master)](https://travis-ci.org/zonyitoo/coio-rs)

Coroutine scheduling with work-stealing algorithm.

## Feature

* Non-blocking I/O
* Work-stealing coroutine scheduling
* Asynchronous computing APIs

## Usage

```toml
[dependencies]
git = "https://github.com/zonyitoo/coio-rs.git"
```

### Basic Coroutines

```rust
extern crate coio;

use coio::Scheduler;

fn main() {
    Scheduler::new()
        .run(|| {
            for _ in 0..10 {
                println!("Heil Hydra");
                Scheduler::sched();
            }
        })
        .unwrap();
}
```

### TCP Echo Server

```rust
extern crate coio;

use std::io::{Read, Write};

use coio::net::TcpListener;
use coio::{spawn, Scheduler};

fn main() {
    // Spawn a coroutine for accepting new connections
    Scheduler::new().with_workers(4).run(move|| {
        let acceptor = TcpListener::bind("127.0.0.1:8080").unwrap();
        println!("Waiting for connection ...");

        for stream in acceptor.incoming() {
            let (mut stream, addr) = stream.unwrap();

            println!("Got connection from {:?}", addr);

            // Spawn a new coroutine to handle the connection
            spawn(move|| {
                let mut buf = [0; 1024];

                loop {
                    match stream.read(&mut buf) {
                        Ok(0) => {
                            println!("EOF");
                            break;
                        },
                        Ok(len) => {
                            println!("Read {} bytes, echo back", len);
                            stream.write_all(&buf[0..len]).unwrap();
                        },
                        Err(err) => {
                            println!("Error occurs: {:?}", err);
                            break;
                        }
                    }
                }

                println!("Client closed");
            });
        }
    }).unwrap();
}
```

## Exit the main function

Will cause all pending coroutine to be killed.

```rust
extern crate coio;

use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

use coio::Scheduler;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));
    let cloned_counter = counter.clone();

    let result = Scheduler::new().run(move|| {
        // Spawn a new coroutine
        Scheduler::spawn(move|| {
            struct Guard(Arc<AtomicUsize>);

            impl Drop for Guard {
                fn drop(&mut self) {
                    self.0.store(1, Ordering::SeqCst);
                }
            }

            let _guard = Guard(cloned_counter);

            coio::sleep_ms(10_000);
            println!("Not going to run this line");
        });

        // Exit right now, which will cause the coroutine to be destroyed.
        panic!("Exit right now!!");
    });

    assert!(result.is_err() && counter.load(Ordering::SeqCst) == 1);
}
```

## Limitations (due to [context-rs](https://github.com/zonyitoo/context-rs))

* Stack size per coroutine is limited (128 KiB by default). You can change this
  limit by using `coio::spawn_opts` or custom `coio::Builder`. A protected page
  is kept at the end of the stack in order to trigger a "Segmentation fault" on
  stack overflow. This is not much of a problem because you'll rarely hit this
  limit unless you have a really deep recursive call, or try to allocate a very
  large slice entirely in stack. (Converting recursive logic into iterative
  code is considered a good practice which libraries usually follow.)

## Basic Benchmarks

See `benchmarks` for more details.
