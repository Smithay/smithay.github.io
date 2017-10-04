Title: About

This is the website of the smithay project, home of several wayland-related rust crates.

The project is named after its flagship crate [smithay](https://github.com/Smithay/smithay), a library for
making wayland compositors in rust, but is also home of [wayland-rs](https://github.com/Smithay/wayland-rs),
a group of low-level bindings to libwayland, as well as a few other crates, either wayland utilities or
needed for smithay, as well as the [Smithay book](https://smithay.github.io/book) aiming to be an in-depth
explanation of wayland's structure and of how smithay and the wayland-rs crates can be used (it's currently
WIP, only containing a good part of the wayland description).

Wayland utilities:

- [wayland-window](https://github.com/Smithay/wayland-window) is a small utility crate for writing wayland
  client apps. It can handle for you the work of drawing the decorations of your windows. These are not
  beautiful and fancy decorations, but they allow basic interaction for the user.
- [wayland-kbd](https://github.com/Smithay/wayland-kbd) is an other small utility for wayland clients. This
  one handles for you the handling of keyboard events and their interpretation with the keymap into unicode
  text.

Smithay dependencies:

- [input.rs](https://github.com/Smithay/input.rs) is a binding crate over the libinput library. It is used
  by smithay to handle raw user input from the kernel (mouse, keyboard and touchscreens).
- [drm-rs](https://github.com/Smithay/drm-rs) is safe interface to the Direct Rendering Manager API, allowing
  Smithay to gain direct access to the monitors of a computer.
- [gbm.rs](https://github.com/Smithay/gbm.rs) is a binding crate over libgbm, the Generic Buffer Manager, which
  allows easier handling of memory buffers in combination with DRM to ease making smithay more portable.