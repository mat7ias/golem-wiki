# gWASM

?> In our gWASM integration, your data and WASM binary is transferred to the remote machine and executed. The target machine is secured by the in-sandbox execution. The binary is portable and compatible with various OS and environments since it is executed by a runtime engine rather than natively. The solution is designed to run large number of computations in parallel in order to profit from Golem Network capabilities.

---

### WASM-Golem applications

We use a standalone [SpiderMonkey](Products/Brass-Beta/gWASM?id=sandboxing) runtime to run WASM binaries. Because of this the applications must be compiled with the Emscripten compiler or compiled to Emscripten target.

Typical gWASM application consists of a client and WASM binary. The client is non-WASM software. Its purpose is to connect to Golem node, create tasks and process results. The example of the client is [g-flite](Products/Brass-Beta/gWASM?id=sample-application). The WASM binary consists of the actual binary .wasm and the JavaScript glue code .js file (both are generated automatically when compiling with Emscripten). It is transferred with input data to provider’s machine and executed by WASM engine. The example is flite compiled to WebAssembly.

---

### Deterministic computations

**Our goal is to make computations fully deterministic.** This way it can be proved that results are correct or incorrect by byte-to-byte comparison. Computations repeated on various machines should always generate the same results. In our solution, we provide redundant computations in order to enable verification of results.

**WebAssembly** is a deterministic machine, but in some points it needs more consideration. **WebAssembly is single threaded by design** - synchronisation and order of execution is not the case. Floating point operations are strict and deterministic. With one exception, NaN representation differs in various environments. There is an execution parameter that enforces determinism but it decreases performance slightly. 

Date and time operations are mocked, you should not rely on them. Currently you cannot access to external devices, like GPU, which are sources of indeterminism. The sandbox emulates pseudo random numbers generation. **Every node draws the same sequence of pseudo random numbers.**

---

### How to build WASM application

Many applications can be compiled to WASM. It is hard to say if a specific code is eligible. That may depend on used syscalls. Sometimes compilation requires some tweaks. If your application just reads data, makes computations and writes results, it is highly likely that it can be compiled to WASM. It is required to install emscripten. Note that WebAssembly is evolving very fast and it is expected to be more adaptive in time. It is also possible to compile Rust source code directly to WASM. See this for more details. 
Again, be sure that you are not violating any licenses or property rights, you take legal responsibility for your actions and it is absolutely fine if you use open source software or your own code.

---

### Limitations

###### 1. Single thread
* All supported applications need to be single threaded. 

###### 2. Once instance
* Forks, IPC calls and synchronization are not allowed. It is convenient to run multiple instances of an application on multi core CPU.

###### 3. Time and date
* You cannot rely on time and date operations. They are mocked for the sake of determinism.

###### 4. CPU only
* All computations are limited to CPU, you cannot access to GPU.

###### 5. Determinism
* You cannot rely on randomness in order to generate cryptography or secrets. Moreover, in order to preserve determinism in the future, we will strive at providing the same source of entropy to all providers involved in verification of a WASM task. This way, the task will have access to real entropy and the determinism on providers’ machines will be preserved.

###### 6. Output size
* All files are mapped to RAM memory. So having input and output files size in total greater than a few GB is not supported. 

---

### Creating WASM tasks in Golem

If you want to run your WASM application (.wasm file and .js file), you can create tasks connecting directly to your Golem node. No additional client, like g-flite, is required. This is pretty straightforward. See this below for more details. Be sure that you are not violating any licenses or property rights: you take legal responsibility for your actions. It is absolutely fine if you use open source software or your own code.


#### Task preparation

The following section describes steps necessary to prepare and create a Wasm
task on Golem.


#### Program compilation

First, you have to compile the code you want to run to WebAssembly using **Emscripteny+JavaScript** backend. The instructions on how to do that can be found
[here](Products/Brass-Beta/gWASM?id=sandboxing).


#### Subtask division

The task is manually divided into subtasks. Each subtask runs the same program, but gets (possibly) different input and execution arguments, and produces (possibly) different output.


#### Input/output

The compiled programs have to read their input from files and write their output to files.

A directory has to be created for the program and its input. The JavaScript and WebAssembly files produced by *Emscripten* have to be placed directly inside this directory. Then, for each subtask, a subdirectory named the same as the subtask has to be created inside the input directory. Everything the program has to access for a particular subtask has to be placed inside its input subdirectory.

Another directory has to be created for program output. The output files specified in `output_file_paths` for each subtask will be copied to a subdirectory named the
same as the subtask inside the output directory.

The final (example) directory structure should look like this:

```bash
.
|-- input_dir
|   |-- program.js
|   |-- program.wasm
|   |-- subtask1
|   |   |-- input_file_1_1
|   |   `-- input_file_1_2
|   `-- subtask2
|       |-- input_file_2_1
|       `-- input_file_2_2
`-- output_dir
    |-- subtask1
    |   |-- output_file_1_1
    |   `-- output_file_1_2
    `-- subtask2
        |-- output_file_2_1
        `-- output_file_2_2
```


#### Task JSON

To create the task, its JSON definition has to be created. The non-task-specific fields that **have** to be present are:

* `type`: has to be `wasm`
* `name`
* `bid`
* `timeout`
* `subtask_timeout`
* `options`: defined below


#### Task options
The following options have to be specified for the WebAssembly task:

* `js_name`: The name of the JavaScript file produced by *Emscripten*. The file should be inside the input directory (specified below).

* `wasm_name`: The name of the WebAssembly file produced by *Emscripten*. The file should be inside the input directory (specified below).

* `input_dir`: The path to the input directory containing the JavaScript and WebAssembly program files and the input subdirectories for each subtask. For each
subtask, its input subdirectory will be mapped to `/` (which is also the *CWD*) inside the program's virtual filesystem.

* `output_dir`: The path to the output directory where for each subtask, the output files specified in `output_file_paths` will be copied to a subdirectory named the 
same as the subtask.

* `subtasks`: A dictionary containing the options for each subtask. The keys should be the subtask names, the values should be dictionaries with fields specified below:

  * `exec_args`: The execution arguments that will be passed to the program for this subtask.

  * `output_file_paths`: The paths to the files the program is expected to produce for this subtask. Each file specified here will be copied from the program's virtual filesystem to the output subdirectory for this subtask. If any of the files are missing, the subtask will fail.
  

#### Example

An example WASM task JSON:

```json
{
    "type": "wasm", 
    "name": "wasm", 
    "bid":  1,
    "subtask_timeout": "00:10:00",
    "timeout": "00:10:00",
    "options": {
        "js_name": "test.js",
        "wasm_name": "test.wasm",
        "input_dir": "/home/user/test_in",
        "output_dir": "/home/user/test_out",
        "subtasks": {
            "subtask1": {
                "exec_args": ["arg1", "arg2"],
                "output_file_paths": ["out.txt"]
            },
            "subtask2": {
                "exec_args": ["arg3", "arg4"],
                "output_file_paths": ["out.txt"]
            }
        }
    }
}
```

#### Creating the task

To create the task, run the following:

```bash
golemcli tasks create path/to/the/task_definition.json
```

---

### Sample application

For the start, please get familiar with our demonstration application - g-flite. This is an **end-to-end and ready to use Golem integration**. All internals and technical details are covered by the client and user friendly interface is exposed for your convenience. This includes creating Golem tasks and completing results. 
You need to run [Golem node](Products/Brass-Beta/Installation) locally, which is currently the default setup for our use cases.


#### g-flite

[![travis-build-status]][travis]

[travis-build-status]: https://travis-ci.org/golemfactory/g-flite.svg?branch=master
[travis]: https://travis-ci.org/golemfactory/g-flite

`g-flite` is a command-line utility which lets you run [flite](http://www.festvox.org/flite/) text-to-speech app on Golem Network.

![g_flite GIF demo](http://i.imgur.com/Ji1CdCN.gif)

?> Note that `g-flite` currently requires that you have [Golem instance](Products/Brass-Beta/Installation) running on the same machine and **only testnet is currently supported** due to the fact that [our WASM platform](https://github.com/golemfactory/sp-wasm) is only available on the testnet.


##### Installation

You can grab a precompiled version of the program for each OS, Linux, Mac, and Win, from [here](https://github.com/golemfactory/g-flite/releases).


##### Building from source

If you wish however, you can also build the program from source. To do this, you'll first need to clone the repo.

```bash
$ git clone --depth 50 https://github.com/golemfactory/g-flite
$ cd g-flite
```

Afterwards, you need to ensure you have Rust installed in version at least `1.34.0`. A good place to get your hands on the latest Rust is [rustup website](https://rustup.rs/).

With Rust installed on your OS, you then need to simply run from within `g-flite` dir

```bash
$ cargo build
```

for debug version, or

```bash
$ cargo build --release
```

for release version. Your program can then be found in

```bash
g-flite/target/debug/g_flite
```

for debug version, or

```bash
g-flite/target/release/g_flite
```

for release version.


##### Usage

Typical usage should not differ much or at all from how you would use the original `flite` app

```bash
$ g_flite some_text_input.txt some_speech_output.wav
```

Note that it is required to specify the name of the output file. All of this assumes that you
have your Golem installed using the default settings

| Setting     | Default value                 |
| ----------- | ----------------------------- |
| datadir     | `$APP_DATA_DIR/golem/default` |
| RPC address | 127.0.0.1                     |
| RPC port    | 61000                         |

`$APP_DATA_DIR` is platform specific:
* on Linux will usually refer to `$HOME/.local/share/<project_path>`
* on Mac will usually refer to `$HOME/Library/Application Support/<project_path>`
* on Windows will usually refer to `{FOLDERID_LocalAppData}/<project_path>/data`

If any of the above information is not correct for your Golem configuration, you can adjust them directly in the command-line as follows

```bash
$ g_flite --address 127.0.0.1 --port 61000 --datadir /abs/path/to/golem/datadir some_text_input.txt some_speech_output.wav
```

Finally, by default `g-flite` will split your input text into 6 subtasks and compute them on Golem Network. You can also adjust this option in the command-line as follows

```bash
$ g_flite --subtasks 2 some_text_input.txt some_speech_output.wav
```

All of this information can also be extracted from the command-line with the `-h` or `--help` flags

```bash
$ g_flite -h

g_flite 0.1.0
Golem RnD Team <contact@golem.network>
flite, a text-to-speech program, distributed over Golem network

USAGE:
    g_flite [FLAGS] [OPTIONS] <TEXTFILE> <WAVFILE>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
    -v, --verbose    Turns verbose logging on

OPTIONS:
        --address <ADDRESS>    Sets RPC address to Golem instance
        --datadir <DATADIR>    Sets path to Golem datadir
        --port <PORT>          Sets RPC port to Golem instance
        --subtasks <NUM>       Sets number of Golem subtasks

ARGS:
    <TEXTFILE>    Input text file
    <WAVFILE>     Output WAV file
```


##### Issues

This program is still very much a work-in-progress, so if you find (and you most likely will) any bugs, please submit them [in our issue tracker](https://github.com/golemfactory/g-flite/issues/new).


##### License

Licensed under [GNU General Public License v3.0](LICENSE) with the exception of `flite` WASM binary which is licensed under [BSD-like License](LICENSE.flite).

---

### Sandboxing

We developed the sandbox and **every WASM application is run within the sandbox on a provider’s machine**. It brings security for providers and allows for better control over execution without sacrificing the performance. This is written in Rust and is available for all common OS. We ship it with Golem client, so users-providers do not need to take any additional steps. For the purpose of tests and development, it is possible to run it locally without Golem client. See below for instructions. The sandbox **includes embedded SpriderMonkey from Mozilla** as the runtime for WASM. Remember that **only Emscripten compiled binaries are supported**.


#### SpiderMonkey-based WebAssembly Sandbox

[![Build Status]][travis] [![Rustc 1.33]][rustc] [![License]][license]

[Build Status]: https://travis-ci.org/golemfactory/sp-wasm.svg?branch=master
[travis]: http://travis-ci.org/golemfactory/sp-wasm
[Rustc 1.33]: https://img.shields.io/badge/rustc-1.33+-lightgray.svg
[rustc]: https://blog.rust-lang.org/2019/02/28/Rust-1.33.0.html
[License]: https://img.shields.io/github/license/golemfactory/sp-wasm.svg 
[license]: https://www.gnu.org/licenses/gpl-3.0.en.html

A WebAssembly sandbox using standalone SpiderMonkey engine. For `v8` version, see [golemfactory/v8-wasm](https://github.com/golemfactory/v8-wasm).

<!-- This WebAssembly sandbox is used in current development version of Golem: [golem/apps/wasm](https://github.com/golemfactory/golem/tree/develop/apps/wasm).
If you would like to launch a gWASM task in Golem, see [here](https://docs.golem.network/#/About/Use-Cases?id=wasm).

- [SpiderMonkey-based WebAssembly Sandbox](#spidermonkey-based-webassembly-sandbox)
  - [Quick start guide](#quick-start-guide)
    - [1. Create and cross-compile simple program](#1-create-and-cross-compile-simple-program)
      - [1.1 C/C++](#11-cc)
      - [1.2 Rust](#12-rust)
    - [2. Create input and output dirs and files](#2-create-input-and-output-dirs-and-files)
    - [3. Run!](#3-run)
  - [Build instructions](#build-instructions)
    - [Using Docker (recommended)](#using-docker-recommended)
    - [Natively on Linux](#natively-on-linux)
    - [Natively on other OSes](#natively-on-other-oses)
  - [CLI arguments explained](#cli-arguments-explained)
  - [Caveats](#caveats)
  - [Wasm store](#wasm-store)
  - [Contributing](#contributing)
  - [License](#license) -->

#### Quick start guide

This guide assumes you have successfully built the `wasm-sandbox` binary; for build instructions, see section [Build instructions](Products/Brass-Beta/gWASM?id=build-instructions) below. If you are running Linux, then you can also use the prebuilt binaries from [here](https://github.com/golemfactory/sp-wasm/releases).


#### 1. Create and cross-compile simple program

Let us create a simple `hello world` style program which will read in some text from `in.txt` text file, read your name from the command line, and save the resultant text in `out.txt`. We'll demonstrate how to cross-compile apps to Wasm for use in Golem in two languages of choice: C and Rust.


#### 1.1 C/C++
```C
#include <stdio.h>

int main(int argc, char** argv) {
  char* name = argc >= 2 ? argv[1] : "anonymous";
  size_t len = 0;
  char* line = NULL;
  ssize_t read;
  
  FILE* f_in = fopen("in.txt", "r");
  FILE* f_out = fopen("out.txt", "w");
  
  while ((read = getline(&line, &len, f_in)) != -1)
      fprintf(f_out, "%s\n", line);
  
  fprintf(f_out, "%s\n", name);
  
  fclose(f_out);
  fclose(f_in);
  
  return 0;
}
```

There is one important thing to notice here. The sandbox communicates the results of computation by reading and writing to files. Thus, every Wasm program is required to at the very least create an output file. If your code does not include file manipulation in its main body, then the Emscripten compiler, by default, will not initialise JavaScript `FS` library, and will trip the sandbox. This will also be true
for programs cross-compiled [from Rust](Products/Brass-Beta/gWASM?id=_12-rust).

Now, we can try and compile the program with Emscripten. In order to do that you need Emscripten SDK installed on your system. For instructions on how to do it, see [here](https://emscripten.org/docs/getting_started/downloads.html).

```
$ emcc -o simple.js -s BINARYEN_ASYNC_COMPILATION=0 simple.c
```

Emscripten will then produce two files: `simple.js` and `simple.wasm`. The produced JavaScript file acts as glue code and sets up all of
the rudimentary syscalls in JavaScript such as `MemFS` (in-memory filesystem), etc., while the `simple.wasm` is our C program cross-compiled to Wasm.

Note here the compiler flag `-s BINARYEN_ASYNC_COMPILATION=0`. By default, the Emscripten compiler enables async IO lib when cross-compiling to Wasm which we currently do not support.
Therefore, in order to alleviate the problem, make sure to always cross-compile with `-s BINARYEN_ASYNC_COMPILATION=0` flag.


#### 1.2 Rust

With Rust, firstly go ahead and create a new binary with `cargo`

```rust
$ cargo new --bin simple
```

Then go ahead and paste the following to `simple/src/main.rs`
file

```rust
use std::env;
use std::fs;
use std::io::{self, Read, Write};

fn main() -> io::Result<()> {
    let args = env::args().collect::<Vec<String>>();
    let name = args.get(1).map_or("anonymous".to_owned(), |x| x.clone());

    let mut in_file = fs::File::open("in.txt")?;
    let mut contents = String::new();
    in_file.read_to_string(&mut contents)?;

    let mut out_file = fs::File::create("out.txt")?;
    out_file.write_all(&contents.as_bytes())?;
    out_file.write_all(&name.as_bytes())?;

    Ok(())
}
```

As was the case with [C program](Products/Brass-Beta/gWASM?id=_11-cc), it is important to notice here that the sandbox communicates the results of computation by reading and writing to files. Thus, every Wasm program is required to at the very least create an output file. If your code does not include file manipulation in its main body, then the Emscripten compiler, by default, will not initialise JavaScript `FS` library, and will trip the sandbox.

In order to cross-compile Rust to Wasm compatible with Golem's sandbox, firstly we need to install the required target which is `wasm32-unknown-emscripten`. The easiest way of doing so, as well as generally managing your Rust installations, is to use [rustup](https://rustup.rs/)

```rust
$ rustup target add wasm32-unknown-emscripten
```

Note that cross-compiling Rust to this target still requires that you have Emscripten SDK installed on your system. For instructions on how to do it, see [here](https://emscripten.org/docs/getting_started/downloads.html).

Now, we can compile our Rust program to Wasm. Make sure you are in the root of your Rust crate, i.e., at the top of `simple` if you didn't change the name of your crate, and run

```rust
$ cargo rustc --target=wasm32-unknown-emscripten --release -- \
  -C link-args="-s BINARYEN_ASYNC_COMPILATION=0"
```

If everything went OK, you should now see two files:

`simple.js` and `simple.wasm` in `simple/target/wasm32-unknown-emscripten/release`.

Just like in [C program](Products/Brass-Beta/gWASM?id=_11-cc)'s case, the produced JavaScript file acts as glue code and sets up all of the rudimentary syscalls in JavaScript such as `MemFS` (in-memory filesystem), etc., while the `simple.wasm` is our Rust program cross-compiled to Wasm.

Again, note here the compiler flag `-s BINARYEN_ASYNC_COMPILATION=0` passed as additional compiler flags to `rustc`. By default, when building for target `wasm32-unknown-emscripten` with `rustc` the compiler will cross-compile with default Emscripten compiler flags which require async IO lib when cross-compiling to Wasm which we currently do not support. Therefore, in order to alleviate the problem, make sure to always cross-compile with `-s BINARYEN_ASYNC_COMPILATION=0` flag.


#### 2. Create input and output dirs and files

The sandbox will require us to specify input and output paths together with output filenames to create, and any additional arguments (see [CLI arguments explained](#cli-arguments-explained) section below for detailed specification of the required arguments). Suppose we have the following file structure locally

```
  |-- in/
  |    |
  |    |-- in.txt
  |
  |-- out/
```

Paste the following text in the `in.txt` file

```
// in.txt
You are running Wasm!
```


#### 3. Run!

After you have successfully run all of the above steps up to now, you should have the following file structure locally

```
  |-- simple.js
  |-- simple.wasm
  |
  |-- in/
  |    |
  |    |-- in.txt
  |
  |-- out/
```

We can now run our Wasm binary inside the sandbox

1. using Docker (if you've followed [Using Docker (recommended)](Products/Brass-Beta/gWASM?id=using-docker-recommended) build instructions)

```
docker run --mount type=bind,source=$PWD,target=/workdir --workdir /workdir \
            wasm-sandbox:latest -I in/ -O out/ -j simple.js -w simple.wasm \
            -o out.txt -- "<your_name>"
```

2. natively (if you're using the prebuilt binaries, or you've built natively following
   [Natively on Linux](#natively-on-linux) build instructions)

```
$ wasm-sandbox -I in/ -O out/ -j simple.js -w simple.wasm \
               -o out.txt -- "<your_name>"
```

Here, `-I` maps the input dir with *all* its contents (files and subdirs) directly to the root `/` in `MemFS`. The output files, on the other hand, will be saved in `out/` local dir. The names of the expected output files have to match those specified with `-o` flags. Thus, in this case, our Wasm bin is expected to create an output file `/out.txt` in `MemFS` which will then be saved in `out/out.txt` locally.

After you execute Wasm bin in the sandbox, `out.txt` should be
created in `out/` dir

```
  |-- simple.js
  |-- simple.wasm
  |
  |-- in/
  |    |
  |    |-- in.txt
  |
  |-- out/
  |    |
  |    |-- out.txt
```

with the contents similar to the following

```
// out.txt
You are running Wasm!
<your_name>
```


#### Build instructions


#### Using Docker (recommended)
To build using Docker, simply run

```bash
$ ./build.sh
```

If you are running Windows, then you can invoke the command in the shell script manually in the command line as follows

```bash
docker build -t wasm-sandbox:latest .
```


#### Natively on Linux

To build natively on Linux, you need to follow the installation instructions of [servo/rust-mozjs](https://github.com/servo/rust-mozjs) and [servo/mozjs](https://github.com/servo/mozjs). The latter is Mozilla's Servo's SpiderMonkey fork and low-level Rust bindings, and as such, requires C/C++ compiler and Autoconf 2.13. See [servo/mozjs/README.md](https://github.com/servo/mozjs) for detailed building instructions.

After following the aforementioned instructions, to build the sandbox, run

```bash
$ cargo build
```

If you would like to build with SpiderMonkey's debug symbols and extensive logging, run instead

```bash
$ cargo build --features "debugmozjs"
```


#### Natively on other OSes

We currently do not offer any support for building the sandbox natively on other OSes.


#### CLI arguments explained

```
wasm-sandbox -I <input-dir> -O <output-dir> -j <wasm-js> -w <wasm> -o <output-file>... -- <args>...
```

where

* `-I` path to the input dir
* `-O` path to the output dir
* `-j` path to the Emscripten JS glue script
* `-w` path to the Emscripten WASM binary
* `-o` paths to expected output files
* `--` anything after this will be passed to the WASM binary as arguments

By default, basic logging is enabled. If you would like to enable more comprehensive logging, export
the following variable

```
RUST_LOG=debug
```


#### Caveats

* If you were following the [Quick start guide](Products/Brass-Beta/gWASM?id=quick-start-guide) you already know that every Wasm bin needs to be cross-compiled by Emscripten with `-s BINARYEN_ASYNC_COMPILATION=0` flag in order to turn off the use of async IO which we currently don't support.

* Sometimes, if the binary you are cross-compiling is of substantial size, you might encounter a `asm2wasm` validation error stating that there is not enough memory assigned to Wasm. In this case, you can circumvent the problem by adding `-s TOTAL_MEMORY=value` flag. The value has to be an integer multiple of 1 Wasm memory page which is currently set at `65,536` bytes.

* When running your Wasm binary you encounter an `OOM` error at runtime, it usually means that the sandbox has run out-of-memory. To alleviate the problem, recompile your program with `-s ALLOW_MEMORY_GROWTH=1`.

* Emscripten, by default, doesn't support `/dev/(u)random` emulation targets different than either browser or `nodejs`. Therefore, we have added basic emulation of the random device that is *fully* deterministic. For details, see [#5](https://github.com/golemfactory/sp-wasm/pull/5).


#### Wasm store

More examples of precompiled Wasm binaries can be found in [golemfactory/wasm-store](Products/Brass-Beta/gWASM?id=store).


#### Contributing to sp-wasm

We welcome all issues and pull requests!

When submitting a pull request, please make sure to run unit tests:

```bash
$ cargo test
```

and integration tests:

```bash
$ cargo build && ./target/debug/sp-wasm-tests
```


#### License

Licensed under [GNU General Public License v3.0](https://github.com/golemfactory/sp-wasm/blob/master/LICENSE).

---

### Store

First of all, we provided a catalog of ready to use gWASM applications (like gflite) in Golem Network here. If you feel that your application is worth to be shared, you can contribute to that also. On the rules see the readme file there. The goal for this catalog is to kick off the adoption and help new users to get familiar with it fast. All apps are self contained but requires to run Golem node locally.  Please see the description to the specific application to know how to use it.


#### Wasm-store

A curated list of precompiled Wasm binaries of programs that are known to successfully work with [Wasm sandbox](https://github.com/golemfactory/sp-wasm) in [Golem](https://github.com/golemfactory/golem).

The list includes applications located directly in this repo, as well as links that point to external sources.

The applications can either be in a raw, Wasm format, or (preferably) they can be augmented with a GUI/CLI for the user's convenience.
Using raw Wasm binaries implies that the user has to be able to prepare the corresponding `task.json` and the required folder structure themselves, and be able to directly connect with their Golem client (e.g., via the use of the [Golem CLI](Products/Brass-Beta/Command-line-interface)). Therefore, as such, this approach requires some technical knowledge of the Golem's internals.
See [here](Products/Brass-Beta/gWASM?id=creating-wasm-tasks-in-golem) to learn how to launch a Wasm task in Golem.

The applications augmented with a GUI/CLI are naturally more user friendly, because they handle communication with Golem node,
Having said that, there currently is no generic way of preparing such a GUI/CLI. There are some examples however. See the [g-flite](Products/Brass-Beta/gWASM?id=g-flite-) app for instance.

The list of applications with GUI/CLI:

* [g-flite](Products/Brass-Beta/gWASM?id=g-flite-) - text-to-speech

The list of raw applications:
* [7-zip](https://github.com/golemfactory/wasm-store/tree/master/7-zip) - 7-zip archiver
* [dcraw](https://github.com/golemfactory/wasm-store/tree/master/dcraw) - raw image to tiff/ppm
* [flite](https://github.com/golemfactory/wasm-store/tree/master/flite) - text-to-speech
* [Minimal Hamiltonian Path](https://github.com/golemfactory/wasm-store/tree/master/MinimalHamiltonianPath) - searches for minimal Hamiltonian path in weighted directed graphs


#### Cloning the repo

When cloning the repo, remember to set up [git-lfs](https://git-lfs.github.com) for this repo on your machine. Usually, this can be accomplished as follows:

```bash
$ git clone https://github.com/golemfactory/wasm-store
$ cd wasm-store
$ git lfs install
$ git lfs pull
```


#### Contributing

We welcome contributions in the form of links to precompiled Wasm binaries of other programs. If you would like to submit such a link, do not hesitate to open a new PR. Your repo should contain README file and license. If it is a raw Wasm binary, it should follow the guidelines below.

For apps augmented with GUI/CLI, the requirements are more relaxed and not set in stone, with the only must-have: good user experience.


#### Directories structure

When contributing an application in a raw Wasm format, please make sure that the submitted link adheres to the structure expected by Wasm task in Golem. That is, we're looking for dir structure similar to the following

```bash
.
|-- task.json
|-- README.md
|-- LICENSE
|-- input_dir
|   |-- program.js
|   |-- program.wasm
|   |-- subtask1
|   |   |-- input_file_1_1
|   |   `-- input_file_1_2
|   `-- subtask2
|       |-- input_file_2_1
|       `-- input_file_2_2
`-- output_dir
    |-- subtask1
    |   |-- output_file_1_1
    |   `-- output_file_1_2
    `-- subtask2
        |-- output_file_2_1
        `-- output_file_2_2
```

where the `task.json` would consist of

```json
{
    "type": "wasm", 
    "name": "program", 
    "bid":  1,
    "subtask_timeout": "00:10:00",
    "timeout": "00:10:00",
    "options": {
        "js_name": "program.js",
        "wasm_name": "program.wasm",
        "input_dir": "<abs_path_to_the_repo>/input_dir",
        "output_dir": "<abs_path_to_the_repo>/output_dir",
        "subtasks": {
            "subtask1": {
                "exec_args": ["arg1_1", "arg1_2"],
                "output_file_paths": ["output_file_1_1", "output_file_1_2"]
            },
            "subtask2": {
                "exec_args": ["arg2_1", "arg2_2"],
                "output_file_paths": ["output_file_2_1", "output_file_2_2"]
            }
        }
    }
}
```
For an example, see for example how [7-zip](7-zip) is set up in this repo.

Of course, if anything is unclear or you find some inconsistencies, please do submit a new issue and we'll make sure it's sorted asap.



