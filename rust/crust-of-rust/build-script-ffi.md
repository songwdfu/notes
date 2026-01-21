## Build script

Buiid script `build/build.rs` is a prog that's run before the crate is built. It
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


