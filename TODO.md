- write Rust structures for `demux_t` and `stream_t`
- make a separate module to import functions from `libvlccore`
- make wrappers for those imported functions. Example: make the `stream_Peek` return a slice
- design a proper library linking system (using cargo, build.rs?)
- is there a nice way (maybe a macro?) to transform `"test"` in `b"test\0".as_ptr()`?
- write wrappers for VLC logging
