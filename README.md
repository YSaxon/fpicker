# fpicker
<img align="right" src="./assets/fpicker_logo.png" alt="Fpicker logo" width="30%"/>

fpicker is a Frida-based fuzzing suite that offers a variety of fuzzing modes for in-process
fuzzing, such as an AFL++ mode or a passive tracing mode. It should run on all platforms that
are supported by Frida.

* [Installation Instructions](#requirements-and-installation)
* [Building and Running](#building-and-running)
* [Creating a Fuzzing Harness](#creating-a-fuzzing-harness)
* [Modes and Configuration](#modes-and-configuration)

Fpicker is based on previous efforts on [ToothPicker](https://github.com/seemoo-lab/toothpicker),
which was developed during my master thesis. Large parts of the development on fpicker were done
during working hours at my employer [ERNW](https://ernw.de).

## Requirements and Installation
Required for running fpicker:
- [frida_compile](https://github.com/frida/frida-compile) to compile the harness script into one JS file
- The `frida-core-devkit` for the respective platform found at [Frida releases on GitHub](https://github.com/frida/frida/releases)
    - depending on the platform you want to target store the library as `frida-core-ios.a`, `frida-core-macos.a`, or `frida-core-linux.a`. 
    Also, linux and macOS/iOS apparently have different header files. For macOS and iOS version `14.0.8` should be used as the built in
    compiler/linker has a bug where CModules cannot be properly linked.

Required only when running in AFL++ mode:
- [AFL++](https://github.com/AFLplusplus/AFLplusplus/)
    - on macOS/iOS:
        - apply the darwin.patch (this disables the check for CrashReporter)
        - on iOS
        - compile with `CFLAGS="-DUSEMMAP=1"`


## Building and Running
Fpicker can be built for `macOS`, `iOS` or `Linux`. The Makefile currently only supports building
for iOS on macOS but it should be totally possible to build fpicker using an iOS toolchain on
Linux.

Depending on the desired target run:

```bash
make fpicker-macos
make fpicker-ios
make fpicker-linux
```

to build fpicker.

Once fpicker is built, the fuzzing harness needs to be built next:

See the [examples folder](./examples) for different sample fuzzing cases. The general approach is as follows:

- Create a custom harness for the target (e.g. `examples/test/test.js`) (see
  [here](#creating-a-fuzzing-harness) for more information on harnesses)
- Compile the custom harness using frida-compile `frida-compile test.js -o harness.js`

Now fpicker can start fuzzing. The exact command highly depends on the configuration and setup. In
the following, a few example cases are given. These mostly correspond to the examples in the
[examples folder](./examples).

- Run fpicker as AFL++ proxy attaching to a target process fuzzing a specific function in process:
```bash
afl-fuzz -i examples/test-network/in -o ./examples/test-network/out -- \\
    ./fpicker --fuzzer-mode afl -e attach -p test-network -f ./examples/test-network/harness.js
```

- Run fpicker in standalone mode attaching to a server and running a client program to send the fuzzing input:
```bash
./fpicker --fuzzer-mode standalone -e attach -p server-process -f harness.js --input-mode cmd \\
    --command "./client-send @@" -i indir -o outdir
```

- Run fpicker in standalone mode attaching to a server, fuzzing in-process with a custom mutator cmd:
```bash
./fpicker --fuzzer-mode active --communication-mode shm -e attach -p server-process -f harness.js \\ 
    -i indir -o outdir --standalone-mutator cmd --mutator-command "radamsa"
```

- Run fpicker in passive mode attaching to a server collecting coverage and payloads:
```bash
./fpicker --fuzzer-mode passive --communication-mode send -e attach -p server-process -o outdir -f harness.js
```


## Creating a Fuzzing Harness
Each target requires its own fuzzing harness. The most important part of this harness is defining
the entry function of Frida's Stalker, which effectively determines at which point the
instrumentation is inserted. In the `in-process` mode this is simple. The function would usually
be the one that is called on each fuzzing iteration. However, it could also be a different one.

A minimalist harness implementation (in `command` mode) could be this:
```javascript
// Import the fuzzer base class
const Fuzzer = require("harness/fuzzer.js");

// The custom fuzzer needs to subclass the Fuzzer class to work properly
class TestFuzzer extends Fuzzer.Fuzzer {
    constructor() {
        // The constructor needs to specify the address of the targeted function and a NativeFunction
        // object that can later be called by the fuzzer.

        const FUZZ_FUNCTION_ADDR = Module.getExportByName(null, "FUZZ_FUNCTION");
        const FUZZ_FUNCTION = new NativeFunction(
            FUZZ_FUNCTION_ADDR,
            "void", ["pointer", "int64"], {
        });

        super("test", FUZZ_FUNCTION_ADDR, FUZZ_FUNCTION);
    }
}

const f = new TestFuzzer();
exports.fuzzer = f;
```
This harness configures the instrumentation to follow the function `FUZZ_FUNCTION`. The
instrumentation will start when this function is entered and stops when the function returns.
This function should be chosen carefully as it is expensive and the more (potentially
unimportant) parts of the process are instrumented, the slower the fuzzer gets. Of course, this is
a consideration between speed and intended coverage. Additionally, the fuzzer currently only
supports functions that are only entered once during one fuzzing iteration, i.e., the function
should not be called more than once during one fuzz case, otherwise the coverage information
might become unreliable.

When the `in-process` mode is used, another function is required in the fuzzer script. The `fuzz`
method. It will get called on each iteration. It will be called with two parameters, a pointer
to a buffer and the length of the buffer. Our exemplary target function takes two parameters, a
pointer to a buffer and its length. Thus, we can just pass the parameters were getting in the
`fuzz` method.

```javascript
fuzz(payload, len) {
    this.target_function(payload, parseInt(len));
}
```

In `passive` mode, a callback needs to be specified that processes the required data. The fuzzer
expects to receive a payload buffer and its length. Depending on the target function that is
fuzzed, this data needs to be extracted. In the following example, we again have a function that
has two parameters: a pointer to a buffer and its length. The `args` parameter contains all
potential parameters the target function receives, so the length parameter (which is the second
one in our case) can be accessed with `args[1]`. We then read the buffer as `Uint8Array` and send
it back to the fuzzer using the `sendPassiveCorpus` method.

```javascript
passiveCallback(args) {
    const len = args[1];
    const data = new Uint8Array(Memory.readByteArray(args[0], parseInt(len)));

    // this encodes the data and sends it back to the fuzzer
    this.sendPassiveCorpus(data, len);
}
```

In case the target needs some sort of preparation before the fuzzer can start, fpicker provides a
`prepare` method that is called during the initialization of the fuzzer. Preparation could be the
establishment of state, e.g., by instantiating an object. Such a preparation function could look
like the following:
```javascript
prepare() {
  // the object can be attached to the fuzzer instance so that it can be used within the
  // fuzz() method later on.
  this.required_object = call_native_function_that_creates_object();
}
```

## Modes and Configuration
pficker offers a large set of modes and configurations that are explained in the following. Most of
these modes can be combined in different ways. At the end of this section is a table that shows
which options can be combined and what their implementation status is.

### Fuzzer Mode
Fpicker has three different *fuzzing modes*: AFL++ Mode, Standalone Active Mode and Standalone
Passive Mode:

- **AFL++ Mode:** In AFL++ mode, fpicker acts as a proxy between AFL++ and the target process. Using
  Frida's instrumentation capabilities, AFL's coverage bitmap is populated while the target is
  fuzzed with input data generated by AFL++.

- **Standalone Active Mode:** In standalone active mode, the fuzzer uses Frida's [Stalker call
  summaries](https://frida.re/docs/javascript-api/#stalker) to gather coverage in form of basic
  blocks that are executed during an iteration. This is nothing new and has been implemented in
  various forms before. However, in combination with some of the other fuzzer settings this can have
  various benefits. It is also a good alternative if AFL++ is not applicable or desired in a given
  environment or case.

- **Standalone Passive Mode:** Passive mode is less of a fuzzer and more of a tracer. Essentially,
  it does the same as standalone active mode. However, it does *not* send its own inputs. It just
  attaches to a certain function and collects coverage. Once new coverage is observed, both the
  coverage and the input is stored.

### Input Mode
While fpicker is largely designed as an in-process fuzzer, it also supports fuzzing via an external
command. For this fpicker offers two input modes.

- **Input Mode In-Process:** In in-process input mode, the harness directly calls a specified
  function in the target process. The fuzzer sends the payload to the harness and the harness
  prepares the payload in such a way that it can call the targeted function.

- **Input Mode CMD:** In command input mode, the payload is redirected to an external command. This
  is useful it is too complex to prepare the parameters other other state when directly calling the
  target function. The coverage collection still needs to be attached to a certain function. Maybe
  there is a client that can be supplied with a payload which then triggers the target function.

### Communication Mode
Communication mode determines how the injected harness communicates with the fuzzer. This largely
depends on the target application. Frida offers an API to send and receive messages from the
injected agent script. This type of communication is quite costly. One of the factors is that the
transported message needs to be encoded in JSON. So sending binary data is straight-forward.
Therefore, fpicker offers a second communicateion mode over shared memory. However, this only works
if it is possible to establish shared memory between the fuzzer and the target application, which
means that this mode cannot be used when the target is attached to the fuzzer host via USB. In CMD
input mode, the communication mode only refers to how the coverage information is communicated back
to the fuzzer, not how the payload is sent, as this is deferred to an external command.

- **Communication Mode Send:** In send communication mode the payload is sent by using Frida's RPC
  calling mechanism. This lets the fuzzer execute a JavaScript function within the injected harness
  script. This function inside the harness can then do all the necessary preparations to call the
  target function. Once the target function is returned from, coverage collection will stop and the
  harness can signal the fuzzer that the iteration is finished. This is done by sending the coverage
  information back to the fuzzer using Frida's send API.

- **Communication Mode SHM:** In SHM communication mode the fuzzer and the harness script
  communicate via shared memory and semaphores. A buffer in shared memory is used to send the
  payload and receive the coverage information. Instead of sending and receiving, the two components
  use waiting and posting to the semaphore. Depending on the system and the target, this introduces
  quite some perfomance gains.  Especially, because the binary payload is written to memory once and
  does not have to be encoded and decoded or copied into other memory locations. Unfortunately, this
  mode sometimes leads to a low stability when running with AFL++. Not sure why, yet.

### Exec Mode
Exec mode can be either *spawn* or *attach*. This is pretty self-explanatory. fpicker can either
attach to a runnning process or spawn a process. One thing that is a major difference between the
two modes is that, should the attached target crash, fpicker will not try to respawn.

### Standalone Mutator
In standalone mode fpicker offers three different input mutation strategies. Nicely put, input
mutation certainly has lots of room for improvement.

- **Standalone Mutator NULL:** This mutator does not mutate the payload and just returns a copy of
  the same payload. Mostly for testing purposes. Otherwise not really useful.

- **Standalone Mutator Rand:** A very bad random mutator. All it does is randomly replace values at
  random locations in the original payload. It does not change the payload length.

- **Standalone Mutator Custom:** This mutator can call an external command to mutate payloads. It
  writes the payload to stdin and receives the mutated payload from stdout. Due to its shallow
  implementation it has quite a performance impact.


