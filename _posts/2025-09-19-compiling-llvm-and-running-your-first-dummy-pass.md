---
layout: post
title: "Compiling LLVM and Running Your First Dummy Pass"
categories: [compilers]
tags: [llvm, llvm-pass, how-to]
---
## Introduction

If you’ve ever been curious about how compilers work under the hood, LLVM is one of the best playgrounds you could ask for. It’s powerful, flexible, and used everywhere from programming languages to GPU drivers. The catch? LLVM is huge, and getting started can feel daunting. 

A great place to begin is with LLVM passes. They are the building blocks of compiler optimizations and program analysis. By creating your own passes and experimenting with the IR, you’ll quickly get hands-on with the core concepts that make LLVM tick. 

This post will take you from building LLVM to running a simple “hello world” pass — your first step into compiler internals.


## Prerequisites and Environment Setup

Before diving in, make sure you have the required tools installed. On Ubuntu/Debian, you can get them with:

```bash
sudo apt -y install gcc g++ git cmake ccache ninja-build zlib1g-dev
```

We’ll also need **Git** for cloning LLVM’s source, and **Ccache** for reducing LLVM build time.



## Step 1: Building LLVM from Source

Follow these steps to clone, configure, and build LLVM (based on the official LLVM guide and our experience):

```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project/ 
git checkout -b llvm-17 llvmorg-17.0.1

mkdir build 
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=ON \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=TRUE \   
      -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \   
      -DLLVM_ENABLE_PROJECTS=clang \   
      -DCMAKE_INSTALL_PREFIX=/opt/llvm \ 
      -B build -S llvm  

cmake --build build -j$(nproc) 
cmake --build build --target check-all -j$(nproc)
```

👉 `check-all` runs LLVM’s test suite. If all tests pass, you’re good to go.
👉 `-DBUILD_SHARED_LIBS` avoid problems when running your pass with `opt`.

To install LLVM to `/opt/llvm`, run: 
```bash
cmake --install build
```

At this point, you have a working LLVM + Clang build.

## Step 2: Writing a Hello World LLVM Pass

Now let’s write a simple **dummy pass**. This pass won’t change the program — it will just print a message whenever it runs.

Create a new file `hellopass.cpp` with the following content:

```cpp
#include "llvm/IR/Function.h"
#include "llvm/IR/PassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {
struct HelloWorldPass : PassInfoMixin<HelloWorldPass> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    errs() << "Hello from: " << F.getName() << "\n";
    return PreservedAnalyses::all();
  }
};
} // namespace

// Pass registration
extern "C" ::llvm::PassPluginLibraryInfo llvmGetPassPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION, "HelloWorldPass", LLVM_VERSION_STRING,
    [](PassBuilder &PB) {
      PB.registerPipelineParsingCallback(
        [](StringRef Name, FunctionPassManager &FPM,
           ArrayRef<PassBuilder::PipelineElement>) {
          if (Name == "hello-world") {
            FPM.addPass(HelloWorldPass());
            return true;
          }
          return false;
        });
    }
  };
}
```

Subsequent posts will go with an in-depth explanation on passes. Just keep in mind that, fundamentally, we have two "sections/responsibilities" in the listing above: The pass logic and its registration. The `run` method is where your logic goes. Here we just print the function name.

## Step 3: Building the Pass

Use CMake to compile the pass as a shared library. Create a `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.13)

project(HelloPass LANGUAGES CXX)  

find_package(LLVM REQUIRED CONFIG) 

include_directories(${LLVM_INCLUDE_DIRS}) 
add_definitions(${LLVM_DEFINITIONS})  
add_library(HelloPass MODULE hellopass.cpp)  
# Ensure C++17 
set_target_properties(HelloPass PROPERTIES CXX_STANDARD 17 
					  CXX_STANDARD_REQUIRED YES)
```

Then build:

```bash
mkdir build && cd build
cmake -DLLVM_DIR=/opt/llvm/lib/cmake/llvm -DCMAKE_CXX_COMPILER='clang++' .. 
cmake --build .
```

This produces `libHelloPass.so`.

## Step 4: Running the Pass

Now, compile a simple C++ file to LLVM IR:

```cpp
// test.cc 
int add(int a, int b) {     
   return a + b; 
}
```

Compile to LLVM IR: 
```bash
/opt/llvm/bin/clang++ -S -emit-llvm test.cc -o test.ll
```

Run the pass:

```bash
/opt/llvm/bin/opt -load-pass-plugin=./libHelloPass.so -passes=hello-world -disable-output < test.ll
```

You should see output like:

```bash
Hello from: add
```

## Wrapping Up

You’ve now:

- Built LLVM from source
- Executed your very first LLVM pass

From here, you can extend the dummy pass to analyze control flow, optimize instructions, or even transform IR. This “Hello World” step is the gateway into real compiler development.