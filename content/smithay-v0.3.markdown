Title: Version 0.3 of Smithay
Date: 2021-07-24 21:00
Category: Releases
Slug: smithay-v-0-3
Authors: Victor Berger
Summary: Announcement of version 0.3 of Smithay, with many improvements and changes since the previous version.

I am proud to finally announce the release of version 0.3 of [Smithay]!

[Smithay] is a library for writing Wayland compositors. A Wayland compositor has a lot of things to manage,
both binding directly to low-level system APIs and managing the numerous Wayland clients that are running
on the system. Smithay provides several abstractions that simplify this job for your, while trying to remain
mostly unopinionated to not constrain the design space for the compositors based on it.

Since the previous release two years and a half ago, Smithay has changed and matured a lot. This blog post
presents the main changes and new stuff that has been done.

## Main fundamental changes

#### Graphics backend rewrite

The graphics backend has largely be rewriten *again* by [**@drakulix**], to completely re-organize it in
a way that better fits the natural handling of client buffers. The structure is now centered around graphics
buffers. The [`backend::allocator`](https://smithay.github.io/smithay/smithay/backend/allocator/index.html)
module provides tools for allocating such buffers using the system libGBM, as well as interoperability with
DMABUF-based buffers submitted by the client apps.

These buffers are to be used in conjunction with the new renderer abstraction defined in the
[`backend::renderer`](https://smithay.github.io/smithay/smithay/backend/renderer/index.html) module,
providing functions to draw the clients' windows content into your main compositor space. A GLES2-based
renderer implementation is also provided, and a Vulkan-based on is on the roadmap.

Alongside this refactor, [Smithay] gained support for DRM atomic modesetting as well as DRM planes.
Furthermore, Anvil is now able to handle multi-monitor setups

#### Client surface state handling

The Wayland protocol mandates intricate semantics for the double-buffered state of wayland surfaces (notably
when considering synchronized subsurfaces), that extend to protocol-extensions that piggy-back on this
mechanism. [Smithay] now provides a generic mechanism for tracking an applying this state. For details about
its usage, see the [`wayland::compositor`](https://smithay.github.io/smithay/smithay/wayland/compositor/index.html)
module, and notably the [`SurfaceData`](https://smithay.github.io/smithay/smithay/wayland/compositor/struct.SurfaceData.html)
struct which is the actual container.

#### Event structure centered on calloop

While previously most backend components of [Smithay] required you to provide a trait-based callback for
handling their events, they are now all converted into being [calloop] event sources. This makes it much
easier to handle your global compositor state, by using
[calloop's shared data mechanism](https://docs.rs/calloop/0.9.0/calloop/index.html), which gives you access
to a `&mut T` reference of your choice in all your callbacks. Anvil uses this pattern: the
[`AnvilState`](https://github.com/Smithay/smithay/blob/7e4e78151aa86df71efb93266205ab3f705f9177/anvil/src/state.rs#L30)
centralizes most of the compositor internal state, allowing us to retain the use of `Rc` and `RefCell` to
minimum.

## New contributors

We are also pleased to have received some significant additions from new contributors, and would like to
thank them for their involvement:

- [**@PolyMeilex**] contributed support for the [`tablet`](https://wayland.app/protocols/tablet-unstable-v2)
  protocol extension for supporting tablet input devices, support for the
  [`xdg-output`](https://wayland.app/protocols/xdg-output-unstable-v1) protocol extension, as well as support
  for [libseat] as a session backend.
- [**@cmeissl**] ensured Smithay can run on `aarch64`-based systems, and improved the handlers for the 
  `xdg-shell` protocol to correctly track the configure states and support popup surfaces. He also diagnosed
  several issues in Smithay and Anvil to the point that anvil can
  [almost run Firefox seamlessly](https://github.com/Smithay/smithay/issues/335).
- [**@psychon**] tested, debugged and improved the XWayland integration, and provided a minimal WM for Anvil,
  allowing it to correctly display and interact with X11 applications.

## Next steps

We now feel that Smithay is in a state where it is realistic to start building  a serious compositor using it.
It is not finished, but the fundamental parts are in place and working.

The next steps we will be taking involve mainly two axes. The first one is introducing support for more Wayland
protocol extensions, mostly from the [wayland-protocols] standard set, but we also consider support for some
protocols from the [wlr-protocols] set. The second axis is the construction of higher-level abstractions for
managing the various parts of the compositor state, and notably the multiple monitors and the organisation of
the client windows and shell interactions in the compositor graphical space.

If you are interested in the project, feel free to drop by in our matrix chatroom `#smithay:matrix.org`,
which is also bridged to IRC at `#smithay` on [libera.chat]. Working on Smithay does not necessarily require
extensive knowledge on the Wayland protocol or the Linux internals, and we are happy to mentor anyone interested
in getting familiar with the project!

[Smithay]: https://crates.io/crates/smithay
[**@drakulix**]: https://github.com/drakulix
[calloop]: https://crates.io/crates/calloop
[libseat]: https://sr.ht/~kennylevinsen/seatd/
[**@PolyMeilex**]: https://github.com/PolyMeilex
[**@cmeissl**]: https://github.com/cmeissl
[**@psychon**]: https://github.com/psychon
[libera.chat]: https://libera.chat/
[wayland-protocols]: https://gitlab.freedesktop.org/wayland/wayland-protocols/
[wlr-protocols]: https://github.com/swaywm/wlr-protocols
