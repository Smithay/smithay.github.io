Title: Version 0.2 of Smithay
Date: 2018-12-28 17:00
Category: Releases
Slug: smithay-v-0-2
Authors: Victor Berger
Summary: Announcement of version 0.2 of smithay, now providing the fundamentals of a wayland compositor.

I am happy to announce the 0.2 release of [smithay](https://crates.io/crates/smithay). Smithay is a library
for writing wayland compositors. With this release, it now contains all the fundamentals for building a
working compositor.

A lot of things have changed since the 0.1 one year ago. A large part of the crate has been completely
rewritten, and work in smithay has also driven evolutions in its dependencies (a large part of which are
also hosted in the smithay project).

## What has changed

Here a quick summary of the changes that occured since 0.1.

#### Anvil

Smithay now comes with `anvil`, a reference implementation of a compositor built on smithay. It
helps a lot with developping and testing new features and reproducing bugs. Anvil can be started
as a Wayland or X11 client (using winit), or directly in a tty.

#### The DRM rewrite

The whole graphics backend has been deeply refactored by [@Drakulix](https://github.com/drakulix),
especially the DRM-handling code and the GBM and EGL abstractions on top of it, which are now clearly
seperated from one another. This improved the structure and clarity of the API, and paved the groundwork
for future integration of new features, like atomic modesetting, EGLStreams, and vulkan support.

#### Full core-protocol support

Smithay now supports the entirety of the core wayland protocol, making it capable of running most
wayland-native apps, such as most programs built on Qt5, Gtk+3 or
[winit](https://crates.io/crates/winit) (including [alacritty](https://github.com/jwilm/alacritty)).

#### Port to wayland-server 0.21

Smithay has been ported to [wayland-server](https://crates.io/crates/wayland-server) version 0.21 and
[calloop](https://crates.io/crates/calloop), which opens the way for use of the pure-rust implementation
of the wayland protocol. Smithay can already run with it, but support for OpenGL clients on the rust
implementation is still blocked, waiting some evolutions of the dma-buf protocol extensions.

#### Basic XWayland support

Smithay is now capable of starting and monitoring the XWayland server, for running X apps under wayland.
However the XWayland server actually also requires to have an X window manager implemented in the wayland
compositor, which Smithay does not have yet.

## The path forward

There is of course still a lot of work to do, but the path forward can now be split into several
more or less independant tasks, now that the core is mostly established. You can have a detailed view
of them in the [issue tracker](https://github.com/smithay/smithay/issues), but here is a quick overview
of the biggest ones:

- Support for input events from touchscreens ([#48](https://github.com/Smithay/smithay/issues/48))
- Implementation of an X Window Manager for XWayland support
  ([#25](https://github.com/Smithay/smithay/issues/25))
- Support for the DMABUF protocol extension ([#96](https://github.com/Smithay/smithay/issues/96))
- Implement classic shell interactions in anvil ([#94](https://github.com/Smithay/smithay/issues/94))
- Write some stand-alone focused examples for smithay ([#99](https://github.com/Smithay/smithay/issues/99))
- Design and introduce higher-level abstractions over the low-level support currently implemented

If you are interested in the project, feel free to drop by in our matrix chatroom
[#smithay:matrix.org](https://matrix.to/#/#smithay:matrix.org), which is also bridged to mozilla's IRC
at *#smithay* and to gitter at [smithay/Lobby](https://gitter.im/smithay/Lobby).

A large part of the work needed on smithay does not require extensive knowledge on the wayland protocol
or the linux internals, and we are happy to mentor anyone interested to get familiar with the project!
