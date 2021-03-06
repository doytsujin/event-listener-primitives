## Event listener primitives

[![Build Status](https://img.shields.io/travis/com/nazar-pc/event-listener-primitives/master?style=flat-square)](https://travis-ci.com/nazar-pc/event-listener-primitives)
[![Crates.io](https://img.shields.io/crates/v/event-listener-primitives?style=flat-square)](https://crates.io/crates/event-listener-primitives)
[![Docs](https://img.shields.io/badge/docs-latest-blue.svg?style=flat-square)](https://docs.rs/event-listener-primitives)
[![License](https://img.shields.io/github/license/nazar-pc/event-listener-primitives?style=flat-square)](https://github.com/nazar-pc/event-listener-primitives)

This crate provides a low-level primitive for building Node.js-like event listeners.

The 2 primitives are `Bag` that is a container for event handlers and `HandlerId` that will remove event handler from the bag on drop.

Close to real-world usage example:
```rust
use event_listener_primitives::{Bag, HandlerId};
use std::sync::Arc;

#[derive(Default)]
struct Handlers {
    bar: Bag<'static, dyn Fn() + Send>,
    closed: Bag<'static, dyn FnOnce() + Send>,
}

struct Inner {
    handlers: Arc<Handlers>,
}

impl Drop for Inner {
    fn drop(&mut self) {
        self.handlers.closed.call_once_simple();
    }
}

#[derive(Clone)]
pub struct Foo {
    inner: Arc<Inner>,
}

impl Foo {
    pub fn new() -> Self {
        let handlers = Arc::<Handlers>::default();

        let inner = Arc::new(Inner { handlers });

        Self { inner }
    }

    pub fn do_bar(&self) {
        // Do things...

        self.inner.handlers.bar.call_simple();
    }

    pub fn on_bar<F: Fn() + Send + 'static>(&self, callback: F) -> HandlerId {
        self.inner.handlers.bar.add(Box::new(callback))
    }

    pub fn on_closed<F: FnOnce() + Send + 'static>(&self, callback: F) -> HandlerId {
        self.inner.handlers.closed.add(Box::new(callback))
    }
}

fn main() {
    let foo = Foo::new();
    let on_bar_handler_id = foo.on_bar(|| {
        println!("On bar");
    });
    foo
        .on_closed(|| {
            println!("On closed");
        })
        .detach();
    // This will trigger "bar" callback just fine since its handler ID is not dropped yet
    foo.do_bar();
    drop(on_bar_handler_id);
    // This will not trigger "bar" callback since its handler ID was already dropped
    foo.do_bar();
    // This will trigger "closed" callback though since we've detached handler OD
    drop(foo);

    println!("Done");
}
```

The output will be:
```text
On bar
On closed
Done
```

## Contribution
Feel free to create issues and send pull requests, they are highly appreciated!

## License
Zero-Clause BSD

https://opensource.org/licenses/0BSD

https://tldrlegal.com/license/bsd-0-clause-license
