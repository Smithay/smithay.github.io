Title: Wayland-rs 0.20 & Smithay's Client Toolkit
Date: 2018-04-26 18:00
Category: Releases
Slug: wayland-rs-v-0-20
Authors: Victor Berger
Summary: Status point of the project: large rework of the wayland bindings in version 0.20, and announcement of Smithay's Client Toolkit.

This article marks the end of a large rework of the
[wayland-rs](https://github.com/smithay/wayland-rs/) group of crates. For context, this
repository hosts a few crates offering bindings to the wayland protocol, both client-side
via [wayland-client](https://docs.rs/wayland-client/) and server-side via
[wayland-server](https://docs.rs/wayland-server/).

This is a major step compared to previous version, 0.14, and the large gap in version numbers
is meant to reflect it. The whole API has been completely overhauled using the insights we
got from working on [smithay](https://github.com/smithay/smithay), using `wayland-server`.

## Summary of the changes

If I had to describe this refactor in a single sentence, I'd say: "don't be afraid to trade
small runtime costs for large ergonomic improvements". The previous design of the libraries
tried to remain as low-overhead possible as possible on the wayland C libs. This caused an
API that was quite convoluted to try to
[make safe sharing of mutable state](https://docs.rs/token_store/), and at 
some points not very different from manually re-implementing a v-table. Other issues more
specific to smithay resulted in a large accumulation of template parameters on types that need
to be named and stored. All these issues made the API quite tedious to use overall.

As a result, for this new version I decided to embrace trait objects, `Rc`, and `Arc`.
Additionnaly, the API redesign was made focusing on *the spirit* of the wayland protocol,
rather than trying to stay close to *the C API*.

To list a few clear gains from this rework:

- The scanner (crate that generate API code from the XML protocol files) almost no longer
  needs to generate `unsafe { }` blocks in its generated code
- The long standing issue of handling both `Send` and non-`Send` shared state is finally
  solved (see next section)
- The ergonomics and flexibility of specifying callbacks for wayland events has been greatly
  improved
- Overall better interaction with objects from the C world, if wayland-rs is used in
  combination with C libraries

## The new callback specification

The most major API change is on the way callbacks are handled. Previously you had to provide
a set of freestanding functions that would be given acces to some state data for each event,
with the difficulties of sharing state between an undeterminate set of callbacks a priori...

Now each time a new protocol object is created, you'll receive it as a `NewProxy`/`NewResource`
object (depending on whether you are client side or server side), and will have to implement
it before accessing the real `Proxy`/`Resource` and being able to use it. Implementing an
object consists in providing a type implementing the appropriate `Implementation<Meta, Msg>`
trait.

This trait has a very simple definition:

```rust
pub trait Implementation<Meta, Msg> {
    fn receive(&mut self, msg: Msg, meta: Meta);
}
```

Here the `Meta` type parameter represent some metadata associated with the message you
receive (often a `Proxy`/`Resource` handle to the wayland object receiving the message),
while `Msg` is a type representing the message itself (often an enum of the possible wayland
messages this object can receive).

The convenience is that `Implementation<Meta, Msg>` is automatically implemented for all
type that implement `FnMut(Msg, Meta)`. As such, you can easily provide closures as
implementations, with all the ergonomics of capturing values and code compacity that
implies. The only catch being that the implementations must be `'static`, so you'll need
to rely on `Arc` or `Rc` to share data. The runtime cost implied should be negligible however,
the wayland protocol part should hardly be a bottleneck for any application.

There are also two ways to implement an object, the default one requiring from the 
implementation to be `Send`, and a secondary one that relaxes this requirement provided you
provide a token that proves you are doing it from the same thread as the one on which the
implementation will be invoked.

## Smithay's Client Toolkit

I also took the occasion of this large API rework to fuse
[wayland-kbd](https://github.com/smithay/wayland-kbd) and
[wayland-window](https://github.com/smithay/wayland-window) into a single crate named
[smithay-client-toolkit](https://github.com/smithay/client-toolkit), which will be some
kind of an equivalent of [smithay](https://github.com/smithay/smithay), but for client
applications. Given the scope of what is possible, it'll clearly remain much simpler than
smithay itself, as it only abstracts the wayland protocol, and nothing like all of
smithay's backends.

Currently, it only provides some barebones functionnalities, but there is still a lot
of room in this toolkit to grow if you want to get involved.

As an example, a minimal client could be created like this:

```rust
extern crate smithay_client_toolkit as sctk;

use sctk::Environment;
use sctk::window::{BasicFrame, Window};

// wayland-client is re-exported in the client toolkit, for convenience
use sctk::reexports::client::{Display, Proxy};
use sctk::reexports::client::protocol::wl_display::RequestsTrait as DisplayRequests;
use sctk::reexports::client::protocol::wl_compositor::RequestsTrait as CompositorRequests;

fn main() {
    // Connect to a wayland server
    let (display, mut event_queue) = Display::connect_to_env().unwrap();

    // The Environment is an abstraction binding for you most of the
    // classic globals. Otherwise this job is very boring and repetitive.
    let env =
        Environment::from_registry(display.get_registry().unwrap(), &mut event_queue).unwrap();

    // Create a wl_surface, which is the canvas on which we can draw the
    // contents of our window
    let surface = env.compositor
        .create_surface()
        .unwrap()
        .implement(|_, _| {
            /* this is the surface implementation, this one ignores all events */
        });
    
    // Now create a Window for this surface
    // The window abstracts for all the protocol handling of the shell, and provides
    // a simplistic decoration for our window (as many wayland compositors require the
    // clients to draw their decorations themselves).
    // The type parameter (here `BasicFrame`) defines what kind of decorations are drawn,
    // and can be customized by implementing the appropriate trait.
    let mut window = Window::<BasicFrame>::init(
        surface,
        (640, 480),
        &env.compositor,
        &env.subcompositor,
        &env.shm,
        &env.shell,
        move |event, ()| {
            /* handle the window's events */
        },
    ).expect("Failed to create a window !");

    // The main event loop
    loop {
        // flush our messages to the server
        display.flush().unwrap();
        // receive and process the messages it sends back
        // this is where all the appropriate implementations are called internally
        event_queue.dispatch().unwrap();

        /*
          Do any other processing we need, including redrawing the surface if needed
        */
    }
}

```