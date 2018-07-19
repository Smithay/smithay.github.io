Title: Wayland-rs 0.21: Pure rust implementation
Date: 2018-07-18 22:00
Category: Releases
Slug: wayland-rs-v-0-21
Authors: Victor Berger
Summary: Announcement of v0.21 of wayland-rs, featuring a pure rust implementation of the wayland protocol.

> [wayland-rs](https://github.com/Smithay/wayland-rs) is a set of crates providing generic APIs to
> manipulate the [Wayland protocol](https://en.wikipedia.org/wiki/Wayland_(display_server_protocol)), 
> successor of X11 for linux windowing.

Here I am finally, after having
[hinted at the possibility](https://github.com/Smithay/wayland-rs/pull/165) and finally taken the
time to write and merge [quite an epic pull request](https://github.com/Smithay/wayland-rs/pull/170),
I can finally say it: wayland-rs is now a pure rust implementation of the protocol, rather than a
crate of bindings to the [wayland system C libraries](https://gitlab.freedesktop.org/wayland/wayland).

Or is it? The people who have already discussed this matter know that abandoning the the C wayland
libraries is really cutting oneself from many interactions. It is most notably required by OpenGL, but also
any other C library that you'd want to use and that interacts directly with wayland like GStreamer for
example. This is obviously not something I'm ready (nor willing) to impose on the users of wayland-rs,
among which are [winit](https://github.com/tomaka/winit) and [glutin](https://github.com/tomaka/glutin),
which quite  obviously need access to OpenGL...

This is why starting from the upcoming 0.21 release, wayland-client and wayland-server will simply let
you choose what you want: the rust implementation or the C one.

This note is organized in three parts: first I'll detail what is the new design of the wayland-rs 
crates and how they can be used, then I'll detail what the future looks like, and finally I'll share
some insights, ideas, thoughts and frustrations I had during my work on this implementation.

### The new organization of wayland-rs

While the API itself did not change a lot from 0.20 to 0.21, there was a very significant refactor
of both wayland-client and wayland-server, shaping them around the new `native_lib` cargo feature.
The deal is simple, by default the crates use the rust implementation, and if you activate the feature
they'll switch to using the system libraries, and expose new parts of the API which give you access
to the necessary C pointers, allowing you to provide them to other C libraries that may need them.

In previous versions, the `native_lib` feature flag was activated by default, and disabling it caused
your program to quickly die on our dear `unimplemented!()` macro. This is no longer the case!

Client side, the `egl` feature of `wayland-client` automatically pulls the `native_lib` feature, as
it is mandatory for OpenGL support and interacting with Mesa. Which means that either you were not
using OpenGL, an you should be able to seamlessly migrate to the rust implementation, or you were
using OpenGL and the crate will just stick to using the system libraries.

### What's next?

The 0.21 version is not yet released. While my implementation passes the few tests I've done manually,
I'm not yet to a coverage and testing I'm confident enough with to make it default. As such I'm only
releasing a `0.21.0-alpha1` version, to test it more thoroughly before making it the default.

Now that this huge milestone is reached, I'm going to radically switch the direction development of
these crates. Up to now, I've mostly focused in making an API as reasonable as possible, but deeply
rooted in the API exposed by the C libs I've been building upon. This will now change, as I plan for
the Rust implementation to be the core drive of design of the crates, and relegate the C backend to
be the one retrofitted into the rust API.

This opens a lot of design space, and as such there are numerous questions I'm not settled yet on.
If you want to take part in this, I'd be glad to hear (or read) what you have to say! I'm
using the [github issue tracker](https://github.com/Smithay/wayland-rs/issues) to keep track of my
questions and ideas. If you first want to get used with the libs, you can help too!
[There are tons of tests that need to be written](https://github.com/Smithay/wayland-rs/tree/master/tests),
and I'm willing to mentor anyone who wants to help.

### The thoughts and insights

The pasts versions of these crates have been an incredible learning experience for me, notably on
the exercise of trying to implement a safe and ergonomic API on top of a C lib that is definitely not
rust-friendly. But actually implementing the protocol itself was quite a new level. I actually 
designed the groundwork for this almost a year ago, and had the branch sleeping on my computer until
I started seriously working on it again 2 months ago. I took some inspiration from
[skylane](https://github.com/perceptia/skylane), but my work was really mostly reverse-engineering the
C libs and taking inspiration from them.

#### The protocol itself

A large design constraint lies in the protocol itself. Wayland is spoken by binary messages over an
unix socket. It is an object oriented protocol, where each message is associated to an object which
defines its interface.

Which means that, to parse a message, you must know the type of the object that sent it. Which means
a large part of the work of the implementation is actually to keep track of the map of which object
exists and what their types are. And given a message can create or destroy objects, you need to have
at least partially processed a message to be able to even *parse* the next one! Talk about being
stateful.

This is quite a constraint on the implementation, and I now better understand how well the callback
oriented approach of the C libraries fits this protocol: the wayland using program provides a large
set of callbacks to the wayland library, one for each possible message, the lib then takes care of
firing them.

#### virtual dispatch and runtime costs

This whole implementation has also been for me quite a wake-up regarding runtime costs. Indeed
Rust has a powerful monomorphisation and a lot can be done statically, but really, sometimes a
trait object does the job pretty well, and at a greater simplicity.

In my case, it would not have been possible to avoid dynamic dispatch anyway: the contents of the
messages the lib receive (which are runtime values) determines which of the (statically-typed)
callbacks should be fired. There is no escape, so I embraced the dynamic dispatch.

As a result, I now have the wayland-scanner parsing the XML specifications of the protocol and its
extensions, generating a lot of rust code, encoding the objects and their interfaces in various traits
with associated types and constants. These traits allow me to build a dynamic representation of the 
objects and their interfaces that the parser can then use at runtime to parse the protocol and 
dispatch the messages to their callbacks.

In particular, I'm doing what I started calling "manual vtables" after a little epiphany I had 
working on that. Because yeah, storing a function pointer in a struct is not very different from
a vtable, is it?

For example I have this `Object` type. It does not has any type parameter linked to the
wayland objects. I can create an `Object` from a type `I` implementing my `Interface` trait:

```rust
let my_object = Object::from_interface::<I>();
```

This is my bridge from the static world to the dynamic one. The fields of the `Object` are filled
with the values from the associated constants of `I`.

And among the fields of `Object` are a few function pointers, like `fn(u16, u32) -> Option<Object>`,
which are initialized by specializing generic functions:

```rust
fn something<I: Interface>(a: u16, b: u32) -> Option<Object> {
    // compute something using a, b, and the associated constants of I
}

let object = Object {
    something_func: something::<MyInterface>,
    ...
}
```

It's not a revolution, but I thought this was an interesting way to use the trait system. My
implementation has a lot of these all over the place, in a deep mix of static and dynamic
dispatch.

To be fair, the C-based code does also have these, so this is not really new. You need to have
a way to retrieve some type information from a C callback where you only provide a function
pointer and a `*mut c_void` data!

#### `Rc`, `Refcell`, `Arc` and `Mutex` are friends, not ennemies

No really, they are all over the place, and they do a pretty good job!

It's cool to be able to avoid runtime costs if you really need to, but sometimes it's not worth
the pain. I don't need top performance, as the wayland socket is hardly a bottleneck in general.

I thus focused on making things work correctly, without any care for performance. I'll come back
later to refactor all that anyway, iterating to improve the API's ergonomics.

Thanks for reading this, and have some happy Rusting!
