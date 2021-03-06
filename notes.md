# Building a VLC module in Rust

a VLC plugin is a DLL linked to libvlccore, exporting the functions `vlc_entry__VERSION`, and optionally `vlc_entry_copyright__VERSION` and `vlc_entry_license__VERSION`.

The C implementation for a module begins with code like this:

```C
static int  Open ( vlc_object_t * );
static void Close( vlc_object_t * );

vlc_module_begin ()
    set_description( N_("WAV demuxer") )
    set_category( CAT_INPUT )
    set_subcategory( SUBCAT_INPUT_DEMUX )
    set_capability( "demux", 142 )
    set_callbacks( Open, Close )
vlc_module_end ()
```

This is a serie of macro calls (defined in `include/vlc_plugin.h`) to generate this entry function:

```C
int vlc_entry__MODULE_NAME (vlc_set_cb, void *); int vlc_entry__MODULE_NAME (vlc_set_cb vlc_set, void *opaque) { module_t *module; module_config_t *config = ((void*)0); if (vlc_set (opaque, ((void*)0), VLC_MODULE_CREATE, &module)) goto error; if (vlc_set (opaque, module, VLC_MODULE_NAME, ("modules/demux/wav.c"))) goto error;·
    if (vlc_set (opaque, module, VLC_MODULE_DESCRIPTION, (const char *)(N_("WAV demuxer")))) goto error;
    vlc_set (opaque, ((void*)0), VLC_CONFIG_CREATE, (0x06), &config); vlc_set (opaque, config, VLC_CONFIG_VALUE, (int64_t)(4));
    vlc_set (opaque, ((void*)0), VLC_CONFIG_CREATE, (0x07), &config); vlc_set (opaque, config, VLC_CONFIG_VALUE, (int64_t)(403));
    if (vlc_set (opaque, module, VLC_MODULE_CAPABILITY, (const char *)("demux")) || vlc_set (opaque, module, VLC_MODULE_SCORE, (int)(142))) goto error;
    if (vlc_set (opaque, module, VLC_MODULE_CB_OPEN, Open) || vlc_set (opaque, module, VLC_MODULE_CB_CLOSE, Close)) goto error;
(void) config; return 0; error: return -1; }
```

With better indentation:

```C
int vlc_entry__MODULE_NAME (vlc_set_cb, void *);
int vlc_entry__MODULE_NAME (vlc_set_cb vlc_set, void *opaque) {
    module_t *module;
    module_config_t *config = ((void*)0);
    if (vlc_set (opaque, ((void*)0), VLC_MODULE_CREATE, &module))
        goto error;
    if (vlc_set (opaque, module, VLC_MODULE_NAME, ("modules/demux/wav.c")))
        goto error;
    if (vlc_set (opaque, module, VLC_MODULE_DESCRIPTION, (const char *)(N_("WAV demuxer"))))
        goto error;
    vlc_set (opaque, ((void*)0), VLC_CONFIG_CREATE, (0x06), &config);
    vlc_set (opaque, config, VLC_CONFIG_VALUE, (int64_t)(4));
    vlc_set (opaque, ((void*)0), VLC_CONFIG_CREATE, (0x07), &config);
    vlc_set (opaque, config, VLC_CONFIG_VALUE, (int64_t)(403));
    if (vlc_set (opaque, module, VLC_MODULE_CAPABILITY, (const char *)("demux")) || vlc_set (opaque, module, VLC_MODULE_SCORE, (int)(142)))
        goto error;
    if (vlc_set (opaque, module, VLC_MODULE_CB_OPEN, Open) || vlc_set (opaque, module, VLC_MODULE_CB_CLOSE, Close))
        goto error;
    (void) config;
    return 0;

  error:
    return -1;
}
```

To create the corresponding function in Rust, we need to export a function receiving a callback with the following definition:

```C
EXTERN_SYMBOL typedef int (*vlc_set_cb) (void *, void *, int, ...);
```

Place the built plugin in `VLC.app/Contents/MacOS/plugins/` (OSX only, of course).
Otherwise, you can see where plugins are stored by looking in the output for "recursively browsing". In my case, this is in "~/vlc/src/.libs/vlc/plugins".

Don't forget to make it end with "_plugin.so" (or "_plugin.dylib" on OSX)! For example, if your file is named "libflv.so", name it "libflv_plugin.so".

To verify that the plugin is loaded, run:

```
./VLC.app/Contents/MacOS/VLC -vvv --list --reset-plugins-cache
```

You should see this on one line: `inrustwetrust          FLV demuxer written in Rust`

To play a file with the new plugin, run:

```
./VLC.app/Contents/MacOS/VLC -vvv --reset-plugins-cache ./zeldaHQ.flv
```

This will first call the `open` function passed as callback.

# Importing functions from libvlccore

copy `VLC.app/Contents/MacOS/lib/libvlccore.8.dylib` to `target/debug/libvlccore.dylib` and build the project.
Update the libvlccore loading path like this:

```
install_name_tool -change "@loader_path/lib/libvlccore.8.dylib" "@loader_path/../lib/libvlccore.8.dylib" target/debug/librustdemux.dylib
```

# Defining the FFI interface

Taking a bit of inspiration from https://github.com/sfackler/rust-openssl :
- the raw structures and C API is (for now) in `src/ffi.rs` and the actual Rust API is in `src/vlc.rs`.
- VLC uses an object-like construction in C, with everything inheriting from `vlc_object_t` through the `VLC_COMMON_MEMBERS` macro
- write wrappers for the C functions to make them more manageable. Example: `stream_Peek` takes a mutable reference to a pointer to a byte array, an expected size, and returns a size (possibly negative in case of error). The Rust-like `stream_Peek` will take a size as argument, and return (if successful) an immutable slice
- all new VLC objects and structures that we do not need to implement will be represented as opaque pointers. Example: `pub type module_t = c_void;`
