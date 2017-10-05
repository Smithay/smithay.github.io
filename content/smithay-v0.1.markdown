Title: Version 0.1 of Smithay
Date: 2017-10-05 17:00
Category: Releases
Slug: smithay-v-0-1
Authors: Victor Berger
Summary: Announcement of version 0.1 of smithay, and a general presentation of the project and its goals.

A few days ago we released version 0.1 of the [smithay crate](https://crates.io/crates/smithay),
its first release on crates.io. On this occasion, let me make a slightly detailed presentation of this
wayland compositor library.

First of all, let me give some context of wayland, in order to show what problem smithay tries to solve.
If you already know about this protocol, feel free to jump to next section.

## Wayland background

The [wayland protocol](https://wayland.freedesktop.org/) aims at replacing the X11 protocol as the standart
communication mean between graphical client apps and the display server on linux. The rationale for this is
that given how X11 is old, has accumulated a lot of debt over the years, and is full of security holes, it's
better to just start over with a new protocol. The project has been going for a few years now and adoption
is getting pretty good. Notably Gtk/Gnome and Qt/KDE are now almost fully compatible, and some linux
distributions have started enabling it by default.

In a few points, here are the main difference between Wayland and X11:

- Wayland is much smaller and barebones. We could push the analogy to the point of saying that Wayland
  is to X11 what Vulkan is to OpenGL. For example, a Wayland client is expected to fully handle the
  drawing of its content, and just hand a buffer to the server via shared memory, while the X11 protocol
  embeds many drawing primitives (though many recent X11 apps already do it that way anyway).
- Wayland puts a lot of focus on security and client isolation. A wayland client does now know if other
  clients are running, let alone accessing their data while any X11 client can hijack the inputs of any
  other, or read the contents of any window. This is typically the kind of things keyloggers do.
- Wayland does not have a single implementation. KDE and Gnome both ship their own wayland server, and there
  is the reference server Weston, as well a some others. X11 had its de-facto unique implementation Xorg,
  and window managers connected to the server as privilegied clients. Wayland window managers *are* the
  servers.

This last point is core to Smithay's use: writing a full wayland server is a lot of work. There is the
need to handle all the clients protocol-wise, do all the job of a window manager, but also manage the
low-level resources with the kernel, initialize the framebuffers, retrieve and process the raw inputs,
do the actual drawing and compositing, draw and manage the whole desktop user interface and so forth…

As such, while big projects like Gnome or KDE can afford doing all this in a very personal and customized
way, smaller wayland compositors have started to rely on compositor libraries that do part of this job
for you. Examples are [libweston](https://github.com/wayland-project/weston),
[wlroots](https://github.com/swaywm/wlroots), or the now mostly abandonned [WLC](https://github.com/Cloudef/wlc).

Smithay is a compositor library written in Rust.

## What's in this 0.1?

This first releases brings the most basic building blocks necessary to build a compositor. Smithay
is designed in a modular bottom-up fashion: we currently have several independant low-level modules,
each managing a very specific part of what "being a wayland compositor" requires. Smithay's user can
then use these modules and configure them as they please to set up their compositor.

We will consider integrating higher-level modules into Smithay in the future. They would be built on
top of the low-level ones, but there is still much work to do with the low-level ones, and this
takes priority!

Smithay modules are currently organized in two groups: backend modules and wayland modules.

#### Backend modules

The backend modules are here to ease integration with the operating system. They abstract away the
setup of the framebuffers, initialization of an OpenGL context to draw on, and retrieval of user
inputs. Currently two backends exist:

- The DRM + libinput backend, which allows to start a smithay-based compositor directly in a TTY,
  like a regular graphical environment.
- the winit backend, which allows to start a smithay-based compositor as a winit app (and as such
  as an X11 or wayland client). Being able to launch the compositor as a client helps a lot with
  the development and debugging.

#### Frontend modules

The wayland modules take care of several aspects of the wayland protocol and manage the
state of the wayland clients. Currently 3 main tasks are implemented:

- Keeping track of shared memory buffers sent by the clients in the form of a shared memory map
  (note: this does not handle opengl clients).
- Keeping track of the client surfaces and their relations to each other. Wayland clients define their
  window contents by creating one or more surfaces, positioning them relative to each other, and
  attaching buffers to them.
- Forwarding inputs to the clients selectively based on current pointer or keyboard focus.

All this together already allows the handling of several type of clients. For example, it is possible to
run `weston-terminal` (a wayland terminal emulator provided by weston) in the smithay examples:

![Smithay running weston-terminal](/images/smithay-0-1.png)

*The black borders are a graphical artefact from smithay's example, that we just haven't got around
fixing yet.*

## How can I run it?

If you want to play around with smithay and see what it can do, check out the
[examples in the github repository](https://github.com/Smithay/smithay/tree/master/examples). A lot
of code is still needed for a compositor built on smithay to work, and there is not nearly enough
space in this blog post to paste the code.

You can also check out [Smithay's documentation on docs.rs](https://docs.rs/smithay). The crate-level
docs are minimalistic, but we try to expand on all the details in the documentation of the individual
modules.

## What's next?

For the next steps, we plan to keep working on the fundamentals. Some of the most important remaining features are:

- Finish implementing the core wayland protocol (still missing: touch events, copy/paste, and a few other things)
- Integrate more with the OS (udev, consolekit/logind) to allow starting proper graphical sessions
- Implement support for OpenGL clients
- Implement support fo Xwayland, the compatibility layer to run X11 apps in a wayland environment

If you are interested, your contributions would be very welcome! Most of the work on the backends don't
require any wayland knowledge. We've also started to work on a [smithay book](https://smithay.github.io/book/).
For now it only contains explanations of the wayland protocol itself, but we plan to incorporate detailed tutorials
on how to create wayland client apps using the `wayland-client` crate, and how to create wayland compositors using
`wayland-server` and `smithay`.

We have a matrix chatroom [@smithay:matrix.org](https://matrix.to/#/#smithay:matrix.org), which
is also bridged to mozilla's IRC at *#smithay* and to gitter at [smithay/Lobby](https://gitter.im/smithay/Lobby).
Feel free to drop by!
