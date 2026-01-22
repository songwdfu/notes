## Build script

Buiid script `build.rs` is a prog that's run before the crate is built. It
should be runnable under all building environments.

Build script have access to the `OUT_DIR` env var that is under `target/`. Build
script output is not printed to stdout/stderr unless build script fails.

In source, the `env!` macro is env var for compile time, using which we can
access files under `OUT_DIR` that is shared with build scripts. 

The `include!` macro reads the content of file (which could be prepared by build
script in `OUT_DIR`) and insert it as rust code at compile time. Using
`cargo-expand`, we can see the result after macro expand.

Build script can communicate with cargo by writing `cargo: ...` to stdout. e.g.
passing linker flags / telling linker to link specific libs / search for some
paths. It can also pass rustc additional feature configs e.g.
`--cfg(openssl_1_1_1_0)`.

Build script by default runs every time build happens, even if no change.
`cargo:rerun-if-changed=PATH` or `cargo:rerun-if-env-changed=PATH` are cargo
instructions that can be specified to only rerun build script sometime.

If a crate links to an external shared library, should declare 
``` 
[package]
links = "foo" 
``` 

In the cargo.toml file, this makes cargo sanity check that only that one crate
is linked against the specified shared lib to avoid dup symbol, etc. Implication
of this is there should be one crate only whose job is only to link against the
shared/static lib and expose all its FFI interfaces. This crate is by convention
named `libxxx-sys`. rustc linklib, setting search path is still needed to
properly link.

The build script of the sys crate first finds the lib by checking env var then sys
path, or build it from source if `CARGO_FEATURE_VENDORED` is set. It then
generates the binding for the rust program. 

`pkg_config` crate is a thin wrapper around unix `pkg-config`, which can give info
about a shared / static lib. e.g. `--libs` gives linking name, `--cflags`
additional gives copiler flags to pass. The rust crate does search and also
generates the cargo instruction metadata to stdout.

## Rust binding

`extern C` and `#[repr(C)]` allows manually writing rust bindings. `extern "C"`
keyword changes the calling convention of the function marked and the func
serves as a declaration. It is inherently unsafe.

`pub enum xxx {}` is the way to represent opaque type which is not turned into a
rust struct. We only pass pointers to it. 

`bindgen` automatically generates these structs and funcs for the lib data and
interfaces.

When `bindgen` upgrades, the gen'd FFI API might change, which requires the
syscrate to have a major update, and all crates that uses the sys crate has to
use the newest sys crate. That's why stable crates checks in the gen'd FFI
manually in `lib.rs`.

`cargo build --verbose` will print the rustc compile command, in which flags
like `-L native=/usr/lib -l sodium` could be seen. Sometimes `pkg_config` emits
the default sys path with `-L` as well. This could cause annoying stuff. opt out
with `pkg_config::Config::print_system_libs(false)`.

The `ctor` crate allows func to be run at load time before `main`, but it's hard
to report error. Another way to use a lib that requires init is to define an
empty struct with init and other methods.

To make C use rust functions, use `extern "C" fn foo` with proper c types like
`std::os::raw::c_int`, and the function pointer would be similar to `&foo as
*fn`. Be careful that rust allocated memory should be freed in rust, and ditto
for C. Also `#[no_mangle]` might be necessary to make the function name be the
same in the final binary, so C could call it correctly.

There's also the `cbindgen` that generates C bindings of the rust library. And
`CXX` crate that allows working btw C++ and Rust.

## Misc

`autocfg` crate detects compiler features in the current rust env and use it
when possible, e.g. check if type is supported, check if trait is there, etc.

`c_int` is sometimes platform-dependent, unlike `u32`.

`MaybeUninit` has unstable features to e.g. convert slice into raw pointer to
first byte and the reverse. 

`Copy` and `Clone`: `Copy` is implicit bit-wise copy, `Clone` is explicit,
customized implementation. `Copy` is inherently `Clone`, which is just `*self`
as it is bitwise copy. `Copy` cannot have `Drop`. `Copy` is not auto trait, must
do `#[derive(Copy, Clone)]`.

`b""` (support ASCII only) str says treat this as a byte string instead of UTF-8
string.


