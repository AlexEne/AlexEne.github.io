---
layout: post
comments: true
---

Most WebAssembly tutorials and examples you will find online focus on using it inside the browser in order to accelerate various functionality of a website or web app.  
However, there is an area where WebAssembly is really powerful but not talked too much about: outside the browser usage scenarios. That is what we'll focus on in this series of posts.

## What is WebAssembly?
Web people are on a roll of giving bad names to things (web-gpu is another example).  
WebAssembly is neither web or assembly, but a bytecode that can be targeted from languages like C++, C#, Rust and others. This means you can write some Rust code, compile it into WebAssembly and run that code in a WebAssembly virtual machine.  

This is powerful because you won't have to deal with garbage collected scripted languages anymore, and essentially use Rust or C++ as your _scripting language_.  WebAssembly enables predictable and stable performance because it doesn't require garbage collection like the usual options (LUA/JavaScript).

It's a relatively new product and there are a lot of rough edges, especially for out-of-browser scenarios. One of the roughest ones in my experience has been documentation for out-of-browser scenarios and this is the reason for my blog posts, to document my findings and hopefully help some people that may be interested in this subject.  

## Why would we want to run WebAssembly outside of a browser?
For out of browser scenarios, one of its main advantage is that it provides system level access without compromising on security. This is done through WASI, the Web Assembly System Interface. [WASI](https://wasi.dev/) is a collection of C-like functions that provide access to functionality such as `fd_read`, `rand`, `fd_write`, threads (WIP), in a safe way.

Here are a few scenarios where you would be able to use web-assembly outside of a browser:
* A scripting language for a video game.
* To run some code with minimal overhead as Fastly/Cloudflare are doing with their compute-at-edge scenarios.
* To run some easy to update code on IoT devices safely and with minimal runtime overhead.  
* Extreamly fast programs in environments where you can't JIT for _reasons_. 

## Prerequisites
For the best experience in this adventure, I suggest using [Visual Studio Code](https://code.visualstudio.com/) as your IDE and install the following extensions: 
* `rust-analyzer`: for autocomplete and other great features.
* `Code-LLDB`: For debugging with LLDB (even works on Windows)
* `WebAssembly by the WebAssembly foundation`: Allows you to disassemble and inspect `.wasm` binaries.

# Choosing a Virtual Machine
First you need a Virtual Machine (VM) that can run your WebAssembly program. This VM needs to be embeddable, so you can add it in your game engine, or what we will call from now on __host program__. There are a few to pick from: [WASM3](https://github.com/wasm3/wasm3), [Wasmtime](https://github.com/bytecodealliance/wasmtime), [WAMR](https://github.com/bytecodealliance/wasm-micro-runtime), and many others. They have various characteristics, such as supporting JIT, using as little memory as possible and so on and you have to choose one one that fits your target platform and scenario.  

It doesn't matter too much what VM you're choosing besides runtime properties, with the exception of debugging. The only VM that allows for a seamless debugging experience that I've found is Wasmtime (this is another one of those rough edges). So even if you don't plan on deploying that anywhere due to other constraints, I suggest using it as the __debug VM__. Whenever you'd want to debug some WASM code you can launch it with Wasmtime.  

# Writing our first WebAssembly program

First, we need to create a new `lib` project:
```bash
cargo new --lib wasm_example
```

In `Cargo.toml` add the following:

```
[lib]
crate-type = ["cdylib"]
```

Now we can edit `lib.rs` and export the following `C` FFI compatible function from it:

```rust
#[no_mangle]
extern "C" fn sum(a: i32, b: i32) -> i32 {
    let s = a + b;
    println!("From WASM: Sum is: {:?}", s);
    s
}
```

This a function that takes two numbers, adds them, then prints the result before returning their sum.  
WebAssembly doesn't define a default function that's executed after a module is loaded, so in the host program you need to get a function by it's signature, and run it (quite similar to how `dlopen`/`dlsym` works).

We expose this `sum` function (and any other functions we want to call from the host VM) as a function that's callable from `C`, using `[#no_mangle]` and `pub extern "C"`. If you're coming here from some WASM for the browser tutorials, you may notice we don't need to use `wasm-bindgen` at all.

## How do we compile it? 
Rust supports two targets for WebAssembly: `wasm32-unknown-unknown` and `wasm32-wasi`. The first one is bare-bones WebAssembly. Think of it like the `[#no-std]` of WebAssembly. It's the kind you'd use for the browser that doesn't assume any system functions are available.  
  
At the other end, `wasm32-wasi` assumes that the VM exposes the `WASI` functionality, allowing a different implementation of the standard library to be used (the implementation that depends on the WASI functions to be available).  

You can take a look at the available implementations for the Rust's stdlib here: [https://github.com/rust-lang/rust/tree/master/library/std/src/sys](https://github.com/rust-lang/rust/tree/master/library/std/src/sys)    
This is the implementation that assumes WASI functions are available to the rust program when running in a WebAssembly VM: [https://github.com/rust-lang/rust/tree/master/library/std/src/sys/wasi](https://github.com/rust-lang/rust/tree/master/library/std/src/sys/wasi).

To comile for wasm32-wasi run:
```bash
# Run this just once
rustup target add wasm32-wasi

# Compile for the wasm32-wasi target.
cargo build --target wasm32-wasi
```

## But how does `println!()` work?
You may have noticed that we're calling `println!()` and expecting the program to work and print to the console, but how does a WebAssembly program knows how to do that?  

This is why we're using `wasm32-wasi`. This target selects for the rust stdlib the version that assumes some functionality to be there (the `WASI` functions). Printing to the console means just writing to a special file descriptor. Most VMs allow that by default so we don't need to do any special settings, besides compiling the correct `wasm32-wasi` target.  

If you have installed the required extensions for vscode, you can now right click on `target/wasm32-wasi/debug/wasm_example.wasm` and select `Show WebAssembly` and you should have a new file open in vscode that looks like this:

```s
(module
  ....
  (type $t15 (func (param i64 i32 i32) (result i32)))
  (import "wasi_snapshot_preview1" "fd_write" (func $_ZN4wasi13lib_generated22wasi_snapshot_preview18fd_write17h6ec13d25aa9fb6acE (type $t8)))
  (import "wasi_snapshot_preview1" "proc_exit" (func $__wasi_proc_exit (type $t0)))
  (import "wasi_snapshot_preview1" "environ_sizes_get" (func $__wasi_environ_sizes_get (type $t2)))
  (import "wasi_snapshot_preview1" "environ_get" (func $__wasi_environ_get (type $t2)))
  (func $_ZN4core3fmt9Arguments6new_v117hb11611244be67330E (type $t9) (param $p0 i32) (param $p1 i32) (param $p2 i32) (param $p3 i32) (param $p4 i32)
    (local $l5 i32) (local $l6 i32) (local $l7 i32) (local $l8 i32) (local $l9 i32) (local $l10 i32)
    global.get $g0
    local.set $l5
  ...
```

This is a `wat` file. `wat` stands for WebAssembly text format. It's kind of like looking at x64/ARM ASM instructions when disassembling a binary, just uglier and harder to understand. I have read that this was because the creators of WebAssembly couldn't decide on a text format so they just left it in this ugly s-expression form.

The import statements here tell us that the WASM program needs the following functions `proc_exit`, `fd_write`, `environ_get`, `environ_sizes_get` to exist in the `wasi_snapshot_preview1` namespace.  
All imported or exported functions from a WebAssembly module require a namespace. `wasi_snapshot_preview1` is the WASI namespace so you can think of it as a reserved namespace for these functions. `println!` needs `wasi_snapshot_preview1::fd_write` to write to stdout.

## The host program
You can pick any VM that has WASI available. I will use Wasmtime because later on I want to show you how to debug WebAssembly and this VM is the only one where debugging works at the moment.

The program loads the wasm binary file from the path: `examples/wasm_example.wasm`.  
This is the file you have previously compiled that you can find in `wasm_example/target/wasm32-wasi/debug/wasm_example.wasm`. __Make sure you move it in the right place before running the host program.__

Here is the full listing of the host VM rust program that initializes the Wasmtime VM, loads the module, links against WASI and loads and executes the exported `sum` function from the WASM module:

```rust
use std::error::Error;
use wasmtime::*;
use wasmtime_wasi::{Wasi, WasiCtx};

fn main() -> Result<(), Box<dyn Error>> {
    // A `Store` is a sort of "global object" in a sense, but for now it suffices
    // to say that it's generally passed to most constructors.
    // let store = Store::default();
    let engine = Engine::new(Config::new().debug_info(true));
    let store = Store::new(&engine);

    // We start off by creating a `Module` which represents a compiled form
    // of our input wasm module. In this case it'll be JIT-compiled after
    // we parse the text format.
    let module = Module::from_file(&engine, "examples/wasm_example.wasm")?;

    // Link the WASI module to our VM. Wasmtime allows us to decide if WASI is present.
    // So we need to load it here, as our module rquires certain functions to be present from the
    // wasi_snapshot_preview1 namespace as seen above.
    // This makes println!() from our WASM program to work. (it uses fd_write).
    let wasi = Wasi::new(&store, WasiCtx::new(std::env::args())?);
    let mut imports = Vec::new();
    for import in module.imports() {
        if import.module() == "wasi_snapshot_preview1" {
            if let Some(export) = wasi.get_export(import.name()) {
                imports.push(Extern::from(export.clone()));
                continue;
            }
        }
        panic!(
            "couldn't find import for `{}::{}`",
            import.module(),
            import.name()
        );
    }
    // After we have a compiled `Module` we can then instantiate it, creating
    // an `Instance` which we can actually poke at functions on.
    let instance = Instance::new(&store, &module, &imports)?;

    // The `Instance` gives us access to various exported functions and items,
    // which we access here to pull out our `answer` exported function and
    // run it.
    let main = instance.get_func("sum")
        .expect("`main` was not an exported function");

    // There's a few ways we can call the `main` `Func` value. The easiest
    // is to statically assert its signature with `get2` (in this case asserting
    // it takes 2 i32 arguments and returns one i32) and then call it.
    let main = main.get2::<i32, i32, i32>()?;

    // And finally we can call our function! Note that the error propagation
    // with `?` is done to handle the case where the wasm function traps.
    let result = main(5, 4)?;
    println!("From host: Answer returned to the host VM: {:?}", result);
    Ok(())
}
```

The `Cargo.toml` of this project needs to have the following dependencies:
```toml
[dependencies]
wasmtime = "0.19"
wasmtime-wasi = "0.19"
anyhow = "1.0.28"
```

Running this with `cargo run` will print the following output:
```
Compiling wasm_host v0.1.0 (wasm_host)
 Finished dev [unoptimized + debuginfo] target(s) in 35.38s
  Running `target\debug\wasm_host.exe`
From WASM: Sum is: 9
From host: Answer returned to the host VM: 9
```

We can observe that the `println!` from the wasm module has correctly printed to the console and that the returned answer is as expected `9`.

## Conclusion
In this post of my WebAssembly Outside the Browser series we've learned how to compile a program for WebAssembly, set-up a host program to load and run a your WASM binary, execute a function exported by the WASM program and put all that together we ended up adding two numbers and printing their result (from both WebAssembly and the host program).  
In the next parts we will touch areas such as debugging, optimizing program size, exposing functions from the host vm to the WASM program and sharing memory between the two VMs. 

## Bonus study materials
Here is the full [WASM specification](https://webassembly.github.io/spec/core/index.html). For me that it's one of the hardest spec that I've ever had to read.  
I would much rather have this spec similar to a CPU user manual (e.g. [VR4300](http://datasheets.chipdb.org/NEC/Vr-Series/Vr43xx/U10504EJ7V0UMJ1.pdf)), rather than it's current form that is forced into some math-y language that, while correct, brings no extra clarity or insight to the reader.  
I strongly think that the concepts described there could have been very well expressed in an easier to understand and parse language and I don't buy the usual excuse that _"Well actually, it's targeted at VM writers, not normal people"_. We should just accept that it's not accessible at all, and we could do way better.

__Some more materials:__  
Videos:  
[Kevin Hoffman: Building a Containerless Future with WebAssembly](https://www.youtube.com/watch?v=vqBtoPJoQOE)  
[Peter Salomonsen: WebAssembly Music](https://www.youtube.com/watch?v=C8j_ieOm4vE)  

Reading material:  
[WebAssembly.org](https://webassembly.org/)  
[Standardizing WASI: A system interface to run WebAssembly outside the web](https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/)  
[Cliff L. Biffle: Making really tiny WebAssembly graphics demos](http://cliffle.com/blog/bare-metal-wasm/)  
[An overview of WebAssembly's historical context](https://labs.imaginea.com/talk-the-nuts-and-bolts-of-webassembly/)



