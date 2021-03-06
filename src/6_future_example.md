# Futures in Rust

We'll create our own `Futures` together with a fake reactor and a simple
executor which allows you to edit, run an play around with the code right here
in your browser.

I'll walk you through the example, but if you want to check it out closer, you
can always [clone the repository][example_repo] and play around with the code
yourself or just copy it from the next chapter.

There are several branches explained in the readme, but two are
relevant for this chapter. The `main` branch is the example we go through here,
and the `basic_example_commented` branch is this example with extensive
comments.

> If you want to follow along as we go through, initialize a new cargo project
> by creating a new folder and run `cargo init` inside it. Everything we write
> here will be in `main.rs`

## Implementing our own Futures

Let's start off by getting all our imports right away so you can follow along

```rust, noplaypen, ignore
use std::{
    future::Future, pin::Pin, sync::{mpsc::{channel, Sender}, Arc, Mutex},
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
    thread::{self, JoinHandle}, time::{Duration, Instant}
};
```

## The Executor

The executors responsibility is to take one or more futures and run them to completion.

The first thing an `executor` does when it gets a `Future` is polling it.

**When polled one of three things can happen:**

- The future returns `Ready` and we schedule whatever chained operations to run
- The future hasn't been polled before so we pass it a `Waker` and suspend it
- The futures has been polled before but is not ready and returns `Pending`

Rust provides a way for the Reactor and Executor to communicate through the `Waker`. The reactor stores this `Waker` and calls `Waker::wake()` on it once
a `Future` has resolved and should be polled again.

**Our Executor will look like this:**

```rust, noplaypen, ignore
// Our executor takes any object which implements the `Future` trait
fn block_on<F: Future>(mut future: F) -> F::Output {

    // the first thing we do is to construct a `Waker` which we'll pass on to
    // the `reactor` so it can wake us up when an event is ready. 
    let mywaker = Arc::new(MyWaker{ thread: thread::current() }); 
    let waker = waker_into_waker(Arc::into_raw(mywaker));

    // The context struct is just a wrapper for a `Waker` object. Maybe in the
    // future this will do more, but right now it's just a wrapper.
    let mut cx = Context::from_waker(&waker);

    // So, since we run this on one thread and run one future to completion
    // we can pin the `Future` to the stack. This is unsafe, but saves an
    // allocation. We could `Box::pin` it too if we wanted. This is however
    // safe since we shadow `future` so it can't be accessed again and will
    // not move until it's dropped.
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // We poll in a loop, but it's not a busy loop. It will only run when
    // an event occurs, or a thread has a "spurious wakeup" (an unexpected wakeup
    // that can happen for no good reason).
    let val = loop {
        
        match Future::poll(pinned, &mut cx) {

            // when the Future is ready we're finished
            Poll::Ready(val) => break val,

            // If we get a `pending` future we just go to sleep...
            Poll::Pending => thread::park(),
        };
    };
    val
}
```

In all the examples you'll see in this chapter I've chosen to comment the code
extensively. I find it easier to follow along that way so I'll not repeat myself
here and focus only on some important aspects that might need further explanation.

Now that you've read so much about `Generators` and `Pin` already this should
be rather easy to understand. `Future` is a state machine, every `await` point
is a `yield` point. We could borrow data across `await` points and we meet the
exact same challenges as we do when borrowing across `yield` points.

> `Context` is just a wrapper around the `Waker`. At the time of writing this
book it's nothing more. In the future it might be possible that the `Context`
object will do more than just wrapping a `Future` so having this extra 
abstraction gives some flexibility.

As explained in the [chapter about generators](./3_generators_pin.md), we use
`Pin` and the guarantees that give us to allow `Futures` to have self
references.

## The `Future` implementation

Futures has a well defined interface, which means they can be used across the
entire ecosystem. 

We can chain these `Futures` so that once a **leaf-future** is
ready we'll perform a set of operations until either the task is finished or we
reach yet another **leaf-future** which we'll wait for and yield control to the
scheduler.

**Our Future implementation looks like this:**

```rust, noplaypen, ignore
// This is the definition of our `Waker`. We use a regular thread-handle here.
// It works but it's not a good solution. It's easy to fix though, I'll explain
// after this code snippet.
#[derive(Clone)]
struct MyWaker {
    thread: thread::Thread,
}

// This is the definition of our `Future`. It keeps all the information we
// need. This one holds a reference to our `reactor`, that's just to make
// this example as easy as possible. It doesn't need to hold a reference to
// the whole reactor, but it needs to be able to register itself with the
// reactor.
#[derive(Clone)]
pub struct Task {
    id: usize,
    reactor: Arc<Mutex<Reactor>>,
    data: u64,
    is_registered: bool,
}

// These are function definitions we'll use for our waker. Remember the
// "Trait Objects" chapter earlier.
fn mywaker_wake(s: &MyWaker) {
    let waker_ptr: *const MyWaker = s;
    let waker_arc = unsafe {Arc::from_raw(waker_ptr)};
    waker_arc.thread.unpark();
}

// Since we use an `Arc` cloning is just increasing the refcount on the smart
// pointer.
fn mywaker_clone(s: &MyWaker) -> RawWaker {
    let arc = unsafe { Arc::from_raw(s) };
    std::mem::forget(arc.clone()); // increase ref count
    RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
}

// This is actually a "helper funtcion" to create a `Waker` vtable. In contrast
// to when we created a `Trait Object` from scratch we don't need to concern
// ourselves with the actual layout of the `vtable` and only provide a fixed
// set of functions
const VTABLE: RawWakerVTable = unsafe {
    RawWakerVTable::new(
        |s| mywaker_clone(&*(s as *const MyWaker)),     // clone
        |s| mywaker_wake(&*(s as *const MyWaker)),      // wake
        |s| mywaker_wake(*(s as *const &MyWaker)),      // wake by ref
        |s| drop(Arc::from_raw(s as *const MyWaker)),   // decrease refcount
    )
};

// Instead of implementing this on the `MyWaker` oject in `impl Mywaker...` we
// just use this pattern instead since it saves us some lines of code.
fn waker_into_waker(s: *const MyWaker) -> Waker {
    let raw_waker = RawWaker::new(s as *const (), &VTABLE);
    unsafe { Waker::from_raw(raw_waker) }
}

impl Task {
    fn new(reactor: Arc<Mutex<Reactor>>, data: u64, id: usize) -> Self {
        Task {
            id,
            reactor,
            data,
            is_registered: false,
        }
    }
}

// This is our `Future` implementation
impl Future for Task {

    // The output for our kind of `leaf future` is just an `usize`. For other
    // futures this could be something more interesting like a byte array.
    type Output = usize;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut r = self.reactor.lock().unwrap();

        // we check with the `Reactor` if this future is in its "readylist"
        // i.e. if it's `Ready`
        if r.is_ready(self.id) {

            // if it is, we return the data. In this case it's just the ID of
            // the task since this is just a very simple example.
            Poll::Ready(self.id)
        } else if self.is_registered {

            // If the future is registered alredy, we just return `Pending`
            Poll::Pending
        } else {

            // If we get here, it must be the first time this `Future` is polled
            // so we register a task with our `reactor`
            r.register(self.data, cx.waker().clone(), self.id);

            // oh, we have to drop the lock on our `Mutex` here because we can't
            // have a shared and exclusive borrow at the same time
            drop(r);
            self.is_registered = true;
            Poll::Pending
        }
    }
}
```

This is mostly pretty straight forward. The confusing part is the strange way
we need to construct the `Waker`, but since we've already created our own
_trait objects_ from raw parts, this looks pretty familiar. Actually, it's
even a bit easier.

We use an `Arc` here to pass out a ref-counted borrow of our `MyWaker`. This
is pretty normal, and makes this easy and safe to work with. Cloning a `Waker`
is just increasing the refcount in this case.

Dropping a `Waker` is as easy as decreasing the refcount. Now, in special
cases we could choose to not use an `Arc`. So this low-level method is there
to allow such cases. 

Indeed, if we only used `Arc` there is no reason for us to go through all the 
trouble of creating our own `vtable` and a `RawWaker`. We could just implement
a normal trait.

Fortunately, in the future this will probably be possible in the standard
library as well. For now, [this trait lives in the nursery][arc_wake], but my
guess is that this will be a part of the standard library after som maturing.

We choose to pass in a reference to the whole `Reactor` here. This isn't normal.
The reactor will often be a global resource which let's us register interests
without passing around a reference.

> ### Why using thread park/unpark is a bad idea for a library
> 
> It could deadlock easily since anyone could get a handle to the `executor thread`
> and call park/unpark on it.
> 
> 1. A future could call `unpark` on the executor thread from a different thread
> 2. Our `executor` thinks that data is ready and wakes up and polls the future
> 3. The future is not ready yet when polled, but at that exact same time the 
> `Reactor` gets an event and calls `wake()` which also unparks our thread.
> 4. This could happen before we go to sleep again since these processes
> run in parallel.
> 5. Our reactor has called `wake` but our thread is still sleeping since it was
> awake already at that point.
> 6. We're deadlocked and our program stops working

> There is also the case that our thread could have what's called a
> `spurious wakeup` ([which can happen unexpectedly][spurious_wakeup]), which
> could cause the same deadlock if we're unlucky.

There are several better solutions, here are some:

  - [std::sync::CondVar][condvar]
  - [crossbeam::sync::Parker][crossbeam_parker]

## The Reactor

This is the home stretch, and not strictly `Future` related, but we need one
to have an example to run.

Since concurrency mostly makes sense when interacting with the outside world (or
at least some peripheral), we need something to actually abstract over this
interaction in an asynchronous way.

This is the `Reactors` job. Most often you'll see reactors in Rust use a library
called [Mio][mio], which provides non blocking APIs and event notification for
several platforms.

The reactor will typically give you something like a `TcpStream` (or any other
resource) which you'll use to create an I/O request. What you get in return is a
`Future`. 

>If our reactor did some real I/O work our `Task` in would instead be represent
>a non-blocking `TcpStream` which registers interest with the global `Reactor`.
>Passing around a reference to the Reactor itself is pretty uncommon but I find
>it makes reasoning about what's happening easier.

Our example task is a timer that only spawns a thread and puts it to sleep for
the number of seconds we specify. The reactor we create here will create a
**leaf-future** representing each timer. In return the Reactor receives a waker
which it will call once the task is finished.

To be able to run the code here in the browser there is not much real I/O we
can do so just pretend that this is actually represents some useful I/O operation
for the sake of this example.


**Our Reactor will look like this:**

```rust, noplaypen, ignore
// This is a "fake" reactor. It does no real I/O, but that also makes our
// code possible to run in the book and in the playground
struct Reactor {

    // we need some way of registering a Task with the reactor. Normally this
    // would be an "interest" in an I/O event
    dispatcher: Sender<Event>,
    handle: Option<JoinHandle<()>>,

    // This is a list of tasks that are ready, which means they should be polled
    // for data.
    readylist: Arc<Mutex<Vec<usize>>>,
}

// We just have two kind of events. An event called `Timeout`
// and a `Close` event to close down our reactor.
#[derive(Debug)]
enum Event {
    Close,
    Timeout(Waker, u64, usize),
}

impl Reactor {
    fn new() -> Self {
        // The way we register new events with our reactor is using a regular
        // channel
        let (tx, rx) = channel::<Event>();
        let readylist = Arc::new(Mutex::new(vec![]));
        let rl_clone = readylist.clone();

        // This `Vec` will hold handles to all the threads we spawn so we can
        // join them later on and finish our programm in a good manner
        let mut handles = vec![];

        // This will be the "Reactor thread"
        let handle = thread::spawn(move || {
            for event in rx {
                let rl_clone = rl_clone.clone();
                match event {

                    // If we get a close event we break out of the loop we're in
                    Event::Close => break,
                    Event::Timeout(waker, duration, id) => {

                        // When we get an event we simply spawn a new thread
                        // which will simulate some I/O resource...
                        let event_handle = thread::spawn(move || {

                            //... by sleeping for the number of seconds
                            // we provided when creating the `Task`.
                            thread::sleep(Duration::from_secs(duration));

                            // When it's done sleeping we put the ID of this task
                            // on the "readylist"
                            rl_clone.lock().map(|mut rl| rl.push(id)).unwrap();

                            // Then we call `wake` which will wake up our
                            // executor and start polling the futures
                            waker.wake();
                        });

                        handles.push(event_handle);
                    }
                }
            }

            // When we exit the Reactor we first join all the handles on
            // the child threads we've spawned so we catch any panics and
            // release any resources.
            for handle in handles {
                handle.join().unwrap();
            }
        });

        Reactor {
            readylist,
            dispatcher: tx,
            handle: Some(handle),
        }
    }

    fn register(&mut self, duration: u64, waker: Waker, data: usize) {

        // registering an event is as simple as sending an `Event` through
        // the channel.
        self.dispatcher
            .send(Event::Timeout(waker, duration, data))
            .unwrap();
    }

    fn close(&mut self) {
        self.dispatcher.send(Event::Close).unwrap();
    }

    // We need a way to check if any event's are ready. This will simply
    // look through the "readylist" for an event macthing the ID we want to
    // check for.
    fn is_ready(&self, id_to_check: usize) -> bool {
        self.readylist
            .lock()
            .map(|rl| rl.iter().any(|id| *id == id_to_check))
            .unwrap()
    }
}

// When our `Reactor` is dropped we join the reactor thread with the thread
// owning our `Reactor` so we catch any panics and release all resources.
// It's not needed for this to work, but it really is a best practice to join
// all threads you spawn.
impl Drop for Reactor {
    fn drop(&mut self) {
        self.handle.take().map(|h| h.join().unwrap()).unwrap();
    }
}
```

It's a lot of code though, but essentially we just spawn off a new thread
and make it sleep for some time which we specify when we create a `Task`.

Now, let's test our code and see if it works. Since we're sleeping for a couple
of seconds here, just give it some time to run.

In the last chapter we have the [whole 200 lines in an editable window](./8_finished_example.md)
which you can edit and change the way you like.

```rust, edition2018
# use std::{
#     future::Future, pin::Pin, sync::{mpsc::{channel, Sender}, Arc, Mutex},
#     task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
#     thread::{self, JoinHandle}, time::{Duration, Instant}
# };
# 
fn main() {
    // This is just to make it easier for us to see when our Future was resolved
    let start = Instant::now();

    // Many runtimes create a glocal `reactor` we pass it as an argument
    let reactor = Reactor::new();

    // Since we'll share this between threads we wrap it in a 
    // atmically-refcounted- mutex.
    let reactor = Arc::new(Mutex::new(reactor));
    
    // We create two tasks:
    // - first parameter is the `reactor`
    // - the second is a timeout in seconds
    // - the third is an `id` to identify the task
    let future1 = Task::new(reactor.clone(), 1, 1);
    let future2 = Task::new(reactor.clone(), 2, 2);

    // an `async` block works the same way as an `async fn` in that it compiles
    // our code into a state machine, `yielding` at every `await` point.
    let fut1 = async {
        let val = future1.await;
        let dur = (Instant::now() - start).as_secs_f32();
        println!("Future got {} at time: {:.2}.", val, dur);
    };

    let fut2 = async {
        let val = future2.await;
        let dur = (Instant::now() - start).as_secs_f32();
        println!("Future got {} at time: {:.2}.", val, dur);
    };

    // Our executor can only run one and one future, this is pretty normal
    // though. You have a set of operations containing many futures that
    // ends up as a single future that drives them all to completion.
    let mainfut = async {
        fut1.await;
        fut2.await;
    };

    // This executor will block the main thread until the futures is resolved
    block_on(mainfut);

    // When we're done, we want to shut down our reactor thread so our program
    // ends nicely.
    reactor.lock().map(|mut r| r.close()).unwrap();
}

# // ============================= EXECUTOR ====================================
# fn block_on<F: Future>(mut future: F) -> F::Output {
#     let mywaker = Arc::new(MyWaker{ thread: thread::current() }); 
#     let waker = waker_into_waker(Arc::into_raw(mywaker));
#     let mut cx = Context::from_waker(&waker);
#     let val = loop {
#         let pinned = unsafe { Pin::new_unchecked(&mut future) };
#         match Future::poll(pinned, &mut cx) {
#             Poll::Ready(val) => break val,
#             Poll::Pending => thread::park(),
#         };
#     };
#     val
# }
# 
# // ====================== FUTURE IMPLEMENTATION ==============================
# #[derive(Clone)]
# struct MyWaker {
#     thread: thread::Thread,
# }
# 
# #[derive(Clone)]
# pub struct Task {
#     id: usize,
#     reactor: Arc<Mutex<Reactor>>,
#     data: u64,
#     is_registered: bool,
# }
# 
# fn mywaker_wake(s: &MyWaker) {
#     let waker_ptr: *const MyWaker = s;
#     let waker_arc = unsafe {Arc::from_raw(waker_ptr)};
#     waker_arc.thread.unpark();
# }
# 
# fn mywaker_clone(s: &MyWaker) -> RawWaker {
#     let arc = unsafe { Arc::from_raw(s).clone() };
#     std::mem::forget(arc.clone()); // increase ref count
#     RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
# }
# 
# const VTABLE: RawWakerVTable = unsafe {
#     RawWakerVTable::new(
#         |s| mywaker_clone(&*(s as *const MyWaker)),     // clone
#         |s| mywaker_wake(&*(s as *const MyWaker)),      // wake
#         |s| mywaker_wake(*(s as *const &MyWaker)),      // wake by ref
#         |s| drop(Arc::from_raw(s as *const MyWaker)),   // decrease refcount
#     )
# };
# 
# fn waker_into_waker(s: *const MyWaker) -> Waker {
#     let raw_waker = RawWaker::new(s as *const (), &VTABLE);
#     unsafe { Waker::from_raw(raw_waker) }
# }
# 
# impl Task {
#     fn new(reactor: Arc<Mutex<Reactor>>, data: u64, id: usize) -> Self {
#         Task {
#             id,
#             reactor,
#             data,
#             is_registered: false,
#         }
#     }
# }
# 
# impl Future for Task {
#     type Output = usize;
#     fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
#         let mut r = self.reactor.lock().unwrap();
#         if r.is_ready(self.id) {
#             Poll::Ready(self.id)
#         } else if self.is_registered {
#             Poll::Pending
#         } else {
#             r.register(self.data, cx.waker().clone(), self.id);
#             drop(r);
#             self.is_registered = true;
#             Poll::Pending
#         }
#     }
# }
# 
# // =============================== REACTOR ===================================
# struct Reactor {
#     dispatcher: Sender<Event>,
#     handle: Option<JoinHandle<()>>,
#     readylist: Arc<Mutex<Vec<usize>>>,
# }
# #[derive(Debug)]
# enum Event {
#     Close,
#     Timeout(Waker, u64, usize),
# }
# 
# impl Reactor {
#     fn new() -> Self {
#         let (tx, rx) = channel::<Event>();
#         let readylist = Arc::new(Mutex::new(vec![]));
#         let rl_clone = readylist.clone();
#         let mut handles = vec![];
#         let handle = thread::spawn(move || {
#             // This simulates some I/O resource
#             for event in rx {
#                 println!("REACTOR: {:?}", event);
#                 let rl_clone = rl_clone.clone();
#                 match event {
#                     Event::Close => break,
#                     Event::Timeout(waker, duration, id) => {
#                         let event_handle = thread::spawn(move || {
#                             thread::sleep(Duration::from_secs(duration));
#                             rl_clone.lock().map(|mut rl| rl.push(id)).unwrap();
#                             waker.wake();
#                         });
# 
#                         handles.push(event_handle);
#                     }
#                 }
#             }
# 
#             for handle in handles {
#                 handle.join().unwrap();
#             }
#         });
# 
#         Reactor {
#             readylist,
#             dispatcher: tx,
#             handle: Some(handle),
#         }
#     }
# 
#     fn register(&mut self, duration: u64, waker: Waker, data: usize) {
#         self.dispatcher
#             .send(Event::Timeout(waker, duration, data))
#             .unwrap();
#     }
# 
#     fn close(&mut self) {
#         self.dispatcher.send(Event::Close).unwrap();
#     }
# 
#     fn is_ready(&self, id_to_check: usize) -> bool {
#         self.readylist
#             .lock()
#             .map(|rl| rl.iter().any(|id| *id == id_to_check))
#             .unwrap()
#     }
# }
# 
# impl Drop for Reactor {
#     fn drop(&mut self) {
#         self.handle.take().map(|h| h.join().unwrap()).unwrap();
#     }
# }
```

I added a debug printout of the events the reactor registered interest for so we can observe
two things:

1. How the `Waker` object looks just like the _trait object_ we talked about in an earlier chapter
2. In what order the events register interest with the reactor

The last point is relevant when we move on the the last paragraph.

## Async/Await and concurrecy

The `async` keyword can be used on functions as in `async fn(...)` or on a
block as in `async { ... }`. Both will turn your function, or block, into a
`Future`.

These `Futures` are rather simple. Imagine our generator from a few chapters
back. Every `await` point is like a `yield` point.

Instead of `yielding` a value we pass in, we yield the result of calling `poll` on
the next `Future` we're awaiting.

Our `mainfut` contains two non-leaf futures which it will call `poll` on. **Non-leaf-futures**
has a `poll` method that simply polls their inner futures and these state machines
are polled until some "leaf future" in the end either returns `Ready` or `Pending`.

The way our example is right now, it's not much better than regular synchronous
code. For us to actually await multiple futures at the same time we somehow need
to `spawn` them so the executor starts running them concurrently.

Our example as it stands now returns this:

```ignore
Future got 1 at time: 1.00.
Future got 2 at time: 3.00.
```

If these `Futures` were executed asynchronously we would expect to see:

```ignore
Future got 1 at time: 1.00.
Future got 2 at time: 2.00.
```

> Note that this doesn't mean they need to run in parallel. They _can_ run in
parallel but there is no requirement. Remember that we're waiting for some
external resource so we can fire off many such calls on a single thread and
handle each event as it resolves.

Now, this is the point where I'll refer you to some better resources for
implementing a better executor. You should have a pretty good understanding of
the concept of Futures by now helping you along the way.

The next step should be getting to know how more advanced runtimes work and
how they implement different ways of running Futures to completion.

[If I were you I would read this next, and try to implement it for our example.](./conclusion.md#building-a-better-exectuor).

That's actually it for now. There as probably much more to learn, this is enough
for today. 

I hope exploring Futures and async in general gets easier after this read and I
do really hope that you do continue to explore further.

Don't forget the exercises in the last chapter 😊.

[mio]: https://github.com/tokio-rs/mio
[arc_wake]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html
[example_repo]: https://github.com/cfsamson/examples-futures
[playground_example]:https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ca43dba55c6e3838c5494de45875677f
[spurious_wakeup]: https://cfsamson.github.io/book-exploring-async-basics/9_3_http_module.html#bonus-section
[condvar]: https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html
[crossbeam_parker]: https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html