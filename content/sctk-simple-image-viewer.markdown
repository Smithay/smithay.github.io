Title: An image viewer using Smithay's Client Toolkit
Date: 2018-05-10 14:00
Category: Tutorials
Slug: sctk-image-viewer
Authors: Victor Berger
Summary: Small tutorial of using Smithay's Client Toolkit by example, building a small image viewer as a Wayland client.

Following [the previous announce of Smithay's Client Toolkit](/wayland-rs-v-0-20.html), this
post aims at giving a general feeling of how this toolkit can be used, and in general how
Wayland client apps are written. To do so, I'll first recap some general information about
how Wayland works, and then do a step by step construction of a simple Wayland image viewer
app.

<center>![The image viewer started in rootson](/images/img_viewer.png)</center>

## Some context: how Wayland works

Wayland as its core is an object-based message-passing protocol. What I mean by that is that
a client app will need to connect to a unix socket on which the Wayland server is listening,
and then, via this socket connection the client will send requests to the server ("I want to
receive pointer inputs", "I want to draw this", etc...), and receive events from the server
("The pointer moved to this location", "the user pressed this key", etc...). These messages
are attached to objects, which are a protocol construct. For example, a client interested in
receiving pointer inputs will create an instance of the `wl_pointer` interface (think class),
and all messages regarding pointer input will be received through this object.

As said previously, all messages are attached to an object, and objects are created or destroyed
by messages. Thus all objects are created by messages of another object, which is then its
parent. This way, we can see the hierarchy of all objects as a tree. At its root there is the
`wl_display`. This object is automatically created when the connection is established, and
represents the connection itself. From the display a client will instantiate the `wl_registry`,
which is the real core of the protocol tree.

Via events associated to this registry, the client will be notified of the existence of various
global objects. These objects can be instantiated from the registry, and are global in the
sense that they represent an underlying resource, and no matter how many times you create
an instance of a global, all these instances represent the same underlying resource.

There are two kinds of globals:

- Those that represent a capability of the Wayland server. For example the `wl_shm` reprensents
  the capability to use shared-memory buffers.
- Those that represent a resource accessible to the compositor. Typically a peripheral (be it
  physical or logical). These can exist more than one time, for example each monitor connected to
  the computer will appear in the registry as a `wl_output` global.

From these globals, the client can create more objects (each global defining a subtree of
objects) and send requests and receive events through them. All Wayland interactions
are done this way.

The protocol is also extensible: the list of possible interfaces as well as their associated
requests and events are defined in XML files. There is one defining the core protocol (which
notably includes the `wl_diplay` and `wl_registry` interfaces) that every Wayland server is
expected to implement. Additional protocol extensions can also be supported, defined in other
protocol XML files. They appear as more globals in the registry that are simply ingored by
clients not supporting these extensions.

With this in mind, let's start writing our app with Smithay's Client Toolkit (that I'll 
abbreviate as SCTK from now on).

## A simple app: image viewer

Our goal here is to build a very simple client: we will start it from command line
giving it an image file as argument, and it will just open a window showing this image.
We'll also make the window resizable, and it will rescale the image appropriately.

### Image loading

The first part or our app is thus loading this image. This is in no way Wayland-specific,
and we'll just use the `image` crate for that:

```rust
extern crate image;

use std::env;

fn main() {
    // First of all, retrieve the path from the program arguments:
    let path = match env::args_os().skip(1).next() {
        Some(p) => p,
        None => {
            println!("USAGE: ./image_wiewer <PATH>");
            return;
        }
    };
    // now, try to open the image
    // the image crate will take care of auto-detecting the file format
    let image = match image::open(&path) {
        Ok(i) => i,
        Err(e) => {
            println!("Failed to open image {}.", path.to_string_lossy());
            println!("Error was: {:?}", e);
            return;
        }
    };
    // We'll need the image in RGBA for drawing it
    let image = image.to_rgba();

    // ...
}

```

That was the easy part, we now have the image loaded in memory. Time to initialize the
Wayland setup.

### Wayland setup

First of all, we need to connect to the Wayland server:

```rust
extern crate smithay-client-toolkit as sctk;
// sctk reexports Wayland-client for convenience
use sctk::reexports::client::Display;

fn main() {
    // ...

    let (display, mut event_queue) = Display::connect_to_env()
                                        .expect("Failed to connect to the Wayland server.");

    // ...
}
```

This `connect_to_env()` function tells Wayland-client to try to connect to a Wayland
server according to the configuration of the environment variables. This is what most
Wayland clients are expected to do.

Now, from this client, we are supposed to retrieve the `wl_registry` and process its
events to retrieve the global list and bind the globals we need. For this, SCTK provides
us with a first utility that does that for us: `sctk::Environment`. It takes a registry
as an argument, wraps it, processes its events and automatically binding the most classic
globals. This will be sufficient for our example.

```rust
use sctk::Environment;
// our display object is actually of type `Proxy<WlDisplay>`, so we need to import
// this extension trait to access methods representing its requests
use sctk::reexports::client::protocol::wl_display::RequestsTrait as DisplayRequests;

fn main() {
    // ..

    // All request method of Wayland objects return a `Result`, as Wayland-client
    // returns an error if you try to send a message on an object that has already
    // been destroyed by a previous message. We know our display has not been
    // destroyed, so we unwrap().
    let registry = display.get_registry().unwrap();

    // The Environment takes a registry and a mutable borrow of the Wayland event
    // queue. It uses the event queue internally to exchange messages with the server
    // to process the registry event.
    // Like all manipulation of the event loop, it can fail (if the connection is
    // unexpectedly lost), and thus returns a Result. This failing would prevent
    // us to continue though, so we unrap().
    let env = Environment::from_registry(registry, &mut event_queue).unwrap();

    // ..
}
```

Once `Environment::from_registry(..)` returns, we get an already-initialized `Environment`
with various Wayland globals already bound, available as public fields. First of all,
we will use the `wl_compositor` global to create a `wl_surface`. The surfaces are the most
fundamental building block of a Wayland app. It describes a rectangle of pixels to which
you can attach a `wl_buffer` defining its contents. A Wayland app is built on surfaces:

- Most often, they can be assigned a role. The most classic one would be "toplevel surface",
  meaning this surface is actually a window in the traditional sense. But surfaces can also
  be used to change the image of the mouse pointer for example.
- They can be defined relative to a parent surface (they are then a "subsurface"), this can
  typically be useful if different parts of your UI are drawn by different means. For example,
  a video player could use GPU-accelerated routines to decode and display the contents of the
  video, and draw the rest of its UI using the CPU. Subsurfaces are also what SCTK uses
  to draw the decorations of your windows, as we'll see later.
- Input information is given relative to surfaces. A Wayland app does not know the absolute location
  of the pointer for example, but if the pointer is on top of one of its surfaces, it knows its
  location relative to it.

```rust
// the extension trait providing the requests for the compositor object
use sctk::reexports::client::protocol::wl_compositor::RequestsTrait as CompositorRequests;

fn main() {
    // ..

    // Use the compositor global to create a new surface
    let surface = env.compositor
        .create_surface()
        .unwrap() // unwrap for the same reasons as before, we know the compositor
                  // is not yet destroyed
        .implement(|_event, _surface| {
            // Here we implement this surface with a closure processing
            // the event
            //
            // Surface events notify us when it enters or leave an output,
            // this is mostly useful to track HiDPI-scaling.
            //
            // We don't do it in this example, so this closure ignores all
            // events by doing nothing.
        });

    // ..
}
```

Once this surface exists, we'll use it as the main surface of our app, the one
on which we'll draw the image we loaded earlier. However, we must first declare that
this surface will be our main surface, by assigning it the "toplevel surface" role.

SCTK provides a convenience type named `Window`, which handles this for us, as well as
a few other things:

- it draws decorations around our surface to make it look like a window (the grey
  borders you saw in the screenshot above)
- it processes most of the shell interaction for us (maximizing/minimizing, etc..)

But it cannot do everything, and requires us to provide a callback to which some
events will be given that we'll process ourselves.

```rust
use sctk::window::{BasicFrame, Event as WEvent, State, Window};

fn main() {
    // ..

    // First of all, this Arc<Mutex<Option<WEvent>>> will store
    // any event from the window that we need to process. We
    // store them and will process them later in the event loop
    // rather that process them directly because in a batch of
    // generated events, often only the last one needs to actually
    // be processed, and some events may render other obsoletes.
    // See the closure a few lines below for details
    let next_action = Arc::new(Mutex::new(None::<WEvent>));

    // We clone the arc to pass one end to the closure
    let waction = next_action.clone();
    // Now we actually create the window. The type parameter `BasicFrame` here
    // specifies the type we want to use to draw the borders. SCTK currently
    // only provide this one, but if you don't like the (arguably ugly)
    // borders it draws, you just need to implement the appropriate trait
    // to create your own.
    let mut window = Window::<BasicFrame>::init(
        surface,           // the wl_surface that serves as the basis of this window
        image.dimensions(),// the initial internal dimensions of the window (so excluding decorations)
        &env.compositor,   // -+
        &env.subcompositor,//  | The Window constructor needs access to these globals
        &env.shm,          //  | to initialize the window
        &env.shell,        // -+
        move |evt, ()| {
            // This is the closure that processes the Window events.
            // There are 3 possible events:
            // - Close: the user requested the window to be closed, we'll then quit
            // - Configure: the server suggested a new state for the window (possibly
            //   a new size if a resize is in progress). We'll likely need to redraw
            //   our contents
            // - Refresh: the frame itself needs to be redrawn. SCTK does not do this
            //   automatically because it has a cost and should only be done in parts
            //   of the event loop where the client actually wants to draw
            // Here we actually only keep the last event received according to a priority
            // order of Close > Configure > Refresh.
            // Indeed, if we received a Close, there is not point drawing anything anymore as
            // we will exit. A new Configure overrides a previous one, and if we received
            // a Configure we will refresh the frame anyway.
            let mut next_action = waction.lock().unwrap();
            // Check if we need to replace the old event by the new one
            let replace = match (&evt, &*next_action) {
                // replace if there is no old event
                (_, &None)
                // or the old event is refresh
                | (_, &Some(WEvent::Refresh))
                // or we had a configure and received a new one
                | (&WEvent::Configure { .. }, &Some(WEvent::Configure { .. }))
                // or the new event is close
                | (&WEvent::Close, _) => true,
                // keep the old event otherwise
                _ => false,
            };
            if replace {
                *next_action = Some(evt);
            }
        },
        // creating the window may fail if the code drawing the frame
        // fails to initialize itself. For BasicFrame this should not happen
        // unless the system is utterly broken, though.
    ).expect("Failed to create a window !");

    // The window needs access to pointer inputs to handle the user interaction
    // with it: resizing/moving the window, or clicking its button.
    // user interaction in Wayland is done via the wl_seat object, which represents
    // a set of pointer/keyboard/touchscreen (some may be absent) that logically
    // represent a single user. Most systems have only one seat, multi-seat configurations
    // are quite exotic.
    // Thus, we just automatically bind the first seat we find.
    let seat = env.manager
        .instantiate_auto::<wl_seat::WlSeat>()
        .unwrap()
        .implement(move |_, _| {});
    // And advertize it to the Window so it knows of it and can process the
    // required pointer events.
    window.new_seat(&seat);

    // ..
}
```

At this point, we have created most of the Wayland objects we'll need for this app.
It's time to jump to the next step:

### Drawing the contents

In here we'll intialize the few more resources we need to do our drawing, and define
a function that redraws our window for a given size, resizing the loaded image if
necessary.

Defining the contents of a surfaces is done in Wayland by first drawing the contents
ourselves, creating a `wl_buffer` referencing this content, and then attaching the
buffer to the surface. The two mains ways of sharing buffers is via OpenGL or using
a shared memory in case of CPU drawing. We'll do the later.

A shared memory pool in Wayland context is just a file (more precisely a tempfile, that
exists only in memory, not on disk) in which the client writes some content and shares the
file descriptor with the server. Then, when creating a buffer, the client specifies that
the buffer refers to a certain set of pixels the file, and the server will just read these
pixels in the shared file to know the contents of the surface this buffer was attached to.

Now, there is an obvious possibility of race if the client writes the file at the same
time as the server reads it. To handle this, the server actually sends an event on the
buffer to notify that it has finished reading the contents associated with the buffer.
And the client is expected to manage the double-buffering accordingly.

For this purpose, SCTK provides two tools. The first is `MemPool`, it is an abstraction
over a shared memory pool, managing the transmission of the shared file to the server.
The second one is `DoubleMemPool`. As you may guess, this holds two `MemPool`. It gives
you access to one of them, and a way to swap them. A small convenience for doing
double-buffering.

Creating our `DoubleMemPool` is quite simple:

```rust
use sctk::utils::DoubleMemPool;

fn main() {
    // ...

    // DoubleMemPool::new() requires access to the wl_shm global, which represents
    // the capability of the server to use shared memory (SHM). All Wayland servers
    // are required to support it.
    let mut pools = DoubleMemPool::new(&env.shm).expect("Failed to create the memory pools.");

    // ...
}
```

We will now write an utility function that will do the drawing itself. Taking a mutable
borrow of a `MemPool`, a reference of our image, and the target dimensions, it will resize
the image to these dimensions, write it to the memory pool, and return a new buffer to
the newly drawn contents. We'll actually manage the double buffering in the event loop.

```rust
extern crate byteorder;
// IO types to write the memory pool
use std::io::{BufWriter, Seek, SeekFrom, Write};
use byteorder::{NativeEndian, WriteBytesExt};

use sctk::reexports::client::NewProxy;
use sctk::reexports::client::protocol::wl_buffer;
use sctk::utils::MemPool;

// The draw function, which draws `base_image` in the provided `MemPool`,
// at given dimensions, and returns a wl_buffer to it.
//
// It simply returns the `NewProxy`, as we'll take care of implementing the
// wl_buffer outside of this function.
//
// If `base_image` is `None`, it'll just draw black contents. This is to
// improve performance during resizing: we need to redraw the window frequently
// so that its dimensions follow the pointer during the resizing, but resizing the
// image is costly and long. So during an interactive resize of the window we'll
// just draw black contents to not feel laggy.
fn draw(
    pool: &mut MemPool,
    base_image: Option<&image::ImageBuffer<image::Rgba<u8>, Vec<u8>>>,
    (buf_x, buf_y): (u32, u32),
) -> NewProxy<wl_buffer::WlBuffer> {
    // First of all, we make sure the pool is big enough to hold our
    // image. We'll write in ARGB8888 format, meaning 4 bytes per pixel.
    // This resize method will only resize the pool if the requested size is bigger
    // than the current size, as Wayland SHM pools are not allowed to shrink.
    //
    // While writing on the file would automatically grow it, we need to advertize the
    // server of its new size, so the call to this method is necessary.
    pool.resize((4 * buf_x * buf_y) as usize)
        .expect("Failed to resize the memory pool.");

    // Now, we can write the contents. MemPool implements the `Seek` and `Write` traits,
    // so we use it directly as a file to write on.

    // First, seek to the beginning, to overwrite our previous content.
    let _ = pool.seek(SeekFrom::Start(0));
    {
        // A sub-scope to limit our borrow of the pool by this BufWriter.
        // This BufWriter will significantly improve our drawing performance,
        // by reducing the number of syscals we do. =)
        let mut writer = BufWriter::new(&mut *pool);
        if let Some(base_image) = base_image {
            // We have an image to draw

            // first, resize it to the requested size. We just use the function provided
            // by the image crate here.
            let image =
                image::imageops::resize(base_image, buf_x, buf_y, image::FilterType::Nearest);

            // Now, we'll write the pixels of the image to the MemPool.
            //
            // We do this in an horribly inneficient maneer, for the sake of simplicity.
            // We'll send pixels to the server in ARGB8888 format (this is one of the only
            // formats that are guaranteed to be supported), but image provides it in
            // RGBA8888, so we need to do the conversion.
            //
            // Aditionnaly, if the image has some transparent parts, we'll blend them into
            // a white background, otherwise the server will draw our window with a
            // transparent background!
            for pixel in image.pixels() {
                // retrieve the pixel values
                let r = pixel.data[0] as u32;
                let g = pixel.data[1] as u32;
                let b = pixel.data[2] as u32;
                let a = pixel.data[3] as u32;
                // blend them
                let r = ::std::cmp::min(0xFF, (0xFF*(0xFF-a) + a*r)/0xFF);
                let g = ::std::cmp::min(0xFF, (0xFF*(0xFF-a) + a*g)/0xFF);
                let b = ::std::cmp::min(0xFF, (0xFF*(0xFF-a) + a*b)/0xFF);
                // write the pixel
                // We use byteorder, as the Wayland protocol explicitly specifies
                // that the pixels must be written in native endianness
                let _ = writer.write_u32::<NativeEndian>(
                    (0xFF << 24) + (r << 16) + (g << 8) + b
                );
            }
        } else {
            // We do not have any image to draw, so we draw black contents
            for _ in 0..(buf_x * buf_y) {
                let _ = writer.write_u32::<NativeEndian>(0xFF000000);
            }
        }
        // Don't forget to flush the writer, to make sure all the contents are
        // indeed written to the file.
        let _ = writer.flush();
    }
    // Now, we create a buffer to the memory pool pointing to the contents
    // we just wrote
    return pool.buffer(
        0,                       // initial offset of the buffer in the pool
        buf_x as i32,            // width of the buffer, in pixels
        buf_y as i32,            // height of the buffer, in pixels
        4 * buf_x as i32,        // stride: number of bytes between the start of two
                                 //   consecutive rows of pixels
        wl_shm::Format::Argb8888,// the pixel format we wrote in
    )
}
```

We now have all the bricks in place to start running our app, time to write the event loop.

### The event loop

As our app has very little room for user input (we handle nothing except the window being
resized), our event loop will also be quite simple. It needs to:

- check the received events to know if we should quit or redraw
- redraw and/or refresh the frame if necessary
- call the methods of the Wayland event queue to ensure proper processing of the messages

```rust

fn main() {
    // ...

    // First, we initialize a few boolean flags that we'll use to track our state:
    // - the window needs to be redrawn
    let mut need_redraw = false;
    // - the frame needs to be refreshed
    let mut need_refresh = false;
    // - have we done the original bootstrap? Depending on the underlying shell
    //   protocol, we may need to wait until we receive a first configure event
    //   before drawing our window. This flag checks if this is done.
    let mut bootstrapped = false;
    // - number of currently free memory pool, which will be shared by the buffer
    //   implementations
    let free_pools = Rc::new(Cell::new(2u32));
    // - are we currently in the process of being resized? (to draw the image or
    //   black content)
    let mut resizing = false;
    // - a QueueToken, we'll need it to implement the buffers
    let token = event_queue.get_token();
    // - the size of our contents
    let mut dimensions = image.dimensions();

    // if our shell does not need to wait for a configure event, we draw right away.
    //
    // Note that this is only the case for the old wl_shell protocol, which is now
    // deprecated. This code is only for compatibility with old servers that do not
    // support the new standard xdg_shell protocol.
    //
    // But if we have made a fallback to wl_shell, we need to draw right away because we'll
    // never receive a configure event if we don't draw something...
    if !env.shell.needs_configure() {
        need_redraw = true;
        bootstrapped = true;
    }

    // We can now actually enter the event loop!
    loop {
        // First, check if any pending action was received by the
        // Window implementation:
        match next_action.lock().unwrap().take() {
            // We received a Close event, just break from the loop
            // and let the app quit
            Some(WEvent::Close) => break,
            // We receive a Refresh event, store that we need to refresh the
            // frame
            Some(WEvent::Refresh) => {
                need_refresh = true;
            }
            // We received a configure event, our action depends on its
            // contents
            Some(WEvent::Configure { new_size, states }) => {
                // the configure event contains a suggested size,
                // if it is different from our current size, we need to
                // update it and redraw
                if let Some((w, h)) = new_size {
                    if dimensions != (w, h) {
                        dimensions = (w, h);
                        need_redraw = true;
                    }
                }
                // Are we currently resizing ?
                // We check if a resizing just started or stopped,
                // because in this case we'll swap between drawing black
                // and drawing the window (or the reverse), and thus we need to
                // redraw
                let new_resizing = states.contains(&State::Resizing);
                if new_resizing != resizing {
                    need_redraw = true;
                }
                resizing = new_resizing;

                // If we have not bootstrapped yet, this is the first configure
                // event, time to draw for the first time.
                if !bootstrapped {
                    bootstrapped = true;
                    need_redraw = true;
                }
            }
            // No event, nothing new to do.
            None => {}
        }

        if need_redraw {
            // We need to redraw, but can only do it if at least one of the
            // memory pools is not currently used by the server. If both are
            // used, we'll keep the `need_redraw` flag to `true` and try again
            // at next iteration of the loop.
            if free_pools.get() > 0 {
                // Draw the contents in the pool and retrieve the buffer
                let new_buffer = draw(
                    pools.pool(),
                    if resizing { None } else { Some(&image) },
                    dimensions
                );
                // We implement this buffer to check when the server sends us the Release
                // event.
                // This method is unsafe for thread-safety reasons, as the callback we provide
                // is not Send/Synd (it uses an Rc<Cell<>>). But our app is not multi-threaded
                // here, so this is fine. =)
                let buffer = unsafe { new_buffer.implement_nonsend(
                    {
                        // get a handle to the free pools counter, for the closure
                        let notify = free_pools.clone();
                        // The closure that will be called when the Release event is
                        // received
                        // - wl_buffer::Event is an enum with a single variant
                        // - we need to annotate the buffer argument because rust's type
                        //   inference is unfortunately not good enough to find it alone
                        move |wl_buffer::Event::Release, buffer: Proxy<wl_buffer::WlBuffer>| {
                            // This buffer has been released by the server. Increment the
                            // count of free pools...
                            notify.set(notify.get()+1);
                            // ... and destroy the buffer
                            buffer.destroy();
                        }
                    },
                    &token
                ) };
                // Now, swap the MemPools, so next time we'll draw on the other.
                // This is double-buffering.
                pools.swap();
                // Register that one more pool is in use.
                free_pools.set(free_pools.get()-1);
                // Attach this new buffer to our surface.
                window.surface().attach(Some(&buffer), 0, 0);
                // We need to tell the server which part of the surface have changed,
                // it uses it to only reload/redraw relevant parts for performance.
                // In our case everything has likely changed so we damage everything.
                //
                // Two ways are possible for notifying this damage:
                if window.surface().version() >= 4 {
                    // If our server is recent enough and supports at least version 4 of the
                    // wl_surface interface, we can specify the damage in buffer coordinates.
                    // This is obviously the best and do that if possible.
                    window.surface().damage_buffer(0, 0, dimensions.0 as i32, dimensions.1 as i32);
                } else {
                    // Otherwise, we fallback to compatilibity mode. Here we specify damage
                    // in surface coordinates, which would have been different if we had drawn
                    // our buffer at HiDPI resolution. We didn't though, so it is ok.
                    window.surface().damage(0, 0, dimensions.0 as i32, dimensions.1 as i32);
                }
                // We resize our frame, so that is is drawn at the right size by SCTK
                window.resize(dimensions.0, dimensions.1);
                // We also refresh it, so that SCTK actually draws it with the new size
                window.refresh();
                // We commit our drawing surface, so that the server atomically applies
                // all the changes we previously did.
                window.surface().commit();
                // We don't need to redraw or refresh anymore =)
                need_redraw = false;
                need_refresh = false;
            }
        } else if need_refresh {
            // If we don't need to redraw but only to refresh, do it
            window.refresh();
            // We still need to commit our surface, as the decorations are drawn as
            // subsurfaces of it.
            window.surface().commit();
            need_refresh = false;
        }

        // These last two calls are necessary for the processing of Wayland messages:
        // - first, flush all our requests to the server on the unix socket
        display.flush().unwrap();
        // - then, retrieve all the incoming messages
        //   this method blocks until a message arrives from the server, and will process
        //   all events by calling the implementation of the target object or each,
        //   and only return once all pending messages have been processed.
        event_queue.dispatch().unwrap();
    }
}
```

Here we are done with our app. You can find the full source of this program
[in the github repo of SCTK](https://github.com/Smithay/client-toolkit/blob/master/examples/image_viewer.rs)
and try to run it in a Wayland compositor.

Thanks for following me to the end, and I hope I managed to give you a good general
idea of how Wayland works, and how SCTK can be used to make Wayland native apps. =)
