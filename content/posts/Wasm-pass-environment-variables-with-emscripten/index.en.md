---
title: "Use environment variables in WebAssembly with Emscripten"
date: 2023-05-08T18:51:19-04:00
author: "Jaswant"
authorLink: "https://jspanchu.dev"
description: "Use environment variables in WebAssembly with Emscripten"
tags: ["WebAssembly", "Emscripten"]
---

It is typical for native C code to poke and probe a shell environment with POSIX `getenv` and `setenv`. How would an existing C program continue to rely upon environment variables in the browser? I've found that JavaScript can preload environment variables expected by a WebAssembly module using the Emscripten toolchain.

## Native code

This example below queries the environment for a way to greet.

```cpp
// main.cpp
#include <cstdlib>
#include <iostream>

int main(int, char*[])
{
  std::cout << std::getenv("GREETING") << '!' << std::endl;
  return 0;
}

```

## Compile with emscripten

The glue code will use an export name called `createModule`, making the wasm initialization look cleaner. We'll use it later in HTML.

```bash
em++ -sWASM=1 -sMODULARIZE=1 -sEXPORT_NAME=createModule "-sEXPORTED_RUNTIME_METHODS=['ENV']" main.cpp -o main.js
```
This command outputs a `main.wasm` binary and `main.js` source file. The `main.js` is the glue code that defines functions to fetch, compile and run WebAssembly instructions from `main.wasm`.

## Setup environment variable in JS

`configuration` maps functions referenced by the emscripten glue layer. The first is an `stdout` handler which pipes text into the developer console.

```html
<!-- index.html -->
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
</head>
<body>
    <script type="text/javascript" src="main.js"></script>
    <script type='text/javascript'>
        // define wasm module configuration.
        let configuration = {
            'print': (text) => { console.info(text); },
            'preRun': [(runtime) => { runtime.ENV.GREETING = 'HI'; }],
        };
        // initialize the wasm module with above configuration.
        createModule(configuration).then((runtime) => {
            console.log("wasm module initialized");
        })
    </script>
</body>
</html>

```

The second entry preloads the environment variable `GREETING="HI"`. Here, `runtime` refers to the wasm runtime. `preRun` is a list of functions called one by one by the glue code right before initializing and starting up the wasm runtime. The `ENV` object contains key-value pairs corresponding to a shell environment.

You can read additional information on `preRun` in the docs - [preRun](https://emscripten.org/docs/api_reference/module.html#Module.preRun).

That was it! This exact method was used to switch between different rendering backends for the VTK WebAssembly example - [ConeMultiBackend](https://discourse.vtk.org/t/guide-how-do-i-use-vtk-wasm-webgpu-experimental-feature/11164).
