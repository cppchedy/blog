---
layout: post
title: "Writing Your First LLVM Plugin Pass: Counting Add Instructions"
categories: [compilers]
tags: [llvm, llvm-pass, how-to]
---
## Introduction 

In my [previous post](https://cppchedy.github.io/blog/compilers/2025/09/19/compiling-llvm-and-running-your-first-dummy-pass.html), we went through the not-so-glamorous part: building LLVM from source and running a pass with `opt`. With that groundwork out of the way, it’s time to move on to the fun part which is writing the passes.

In this post, we’ll start with a bit of theory on LLVM passes, just enough to give you solid footing, and then jump straight into code. Our first real pass will be a simple one: counting the number of `add` instructions across an IR module.

Since we’ll be working directly with LLVM IR, I’m assuming you’re at least somewhat familiar with reading it. If not, I recommend taking a little time to get comfortable reading IR first because it will make your LLVM adventure much smoother.

## LLVM Passes and Why They Matter

If I had to pick two pillars of LLVM, they would be the **Intermediate Representation (IR)** and the **pass system**.

- **IR** provides a common, language-agnostic format that higher-level languages can target. This abstraction makes LLVM language-independent and enables all subsequent transformations and optimizations.
- **Passes** break down the complex optimization process into small, focused steps. Each pass has a clear task, and LLVM executes them in a pipeline to gradually refine or analyze the IR.

Together, these two ideas form the backbone of LLVM’s flexibility and power. With passes, LLVM becomes a modular framework where analyses and optimizations can be added, removed, or combined. Without them, LLVM would be an unmanageable monolith.

LLVM organizes passes according to the **granularity of the IR unit they operate on** or in simpler term _scope_:

- **Module passes**
- **Function passes**
- **Loop passes**

Let’s break down the different types of passes in more detail.

### Module Pass

- **Scope:** Operates on an entire `Module` (representing a program or translation unit).
- **Use cases:**
    - Interprocedural optimizations (e.g., inlining decisions).
    - Whole-program analyses (e.g., building call graphs).
    - Global variable optimizations.
- **Example:** _Dead Global Elimination_ removes unused global variables.

### Function Pass

- **Scope:** Operates on a single `Function` independently of others.
- **Use cases:**
    - Local optimizations inside a function.
    - Instruction simplification.
    - Control-flow restructuring.
- **Example:** _Instruction Combining (InstCombine)_ simplifies redundant instructions within a function.

### Loop Pass

- **Scope:** Operates on a single `Loop` (as identified by LLVM’s loop analysis).
- **Use cases:**
    - Loop-invariant code motion (LICM).
    - Loop unrolling, peeling, or fusion.
    - Vectorization.
- **Example:** _Loop Unroll Pass_ duplicates loop bodies to reduce loop overhead.

You may want to read about the use cases you find interesting elsewhere; though, maybe I will write about some of them. Next, let's take a look at the Pass Manager.

## Pass Manager Machination

Passes don’t execute on their own. They must be scheduled and orchestrated. That’s the job of the **Pass Manager**, which arranges passes, manages dependencies, and helps preserve analysis results when possible to avoid redundant recomputation.

Before a pass can run, it has to be registered so that LLVM’s PassBuilder and PassManager know it exists and where it can fit in the compilation pipeline. Once registered, the infrastructure can include the pass in a pipeline, deciding when it runs and alongside which other passes.

Keep in mind that analysis preservation isn’t automatic. A pass must explicitly declare which analyses remain valid after it runs. Likewise, different kinds of passes (module, function, loop) are managed at different levels of granularity, and you may need adapters or careful placement when mixing them in a pipeline.

With those ideas in place, let’s move from the conceptual side to the practical one and see how all these pieces come together in code.


## Getting our hands dirty

### Template For Passes

In the previous sections, we discussed LLVM's pass system and briefly introduced the new Pass Manager that LLVM relies on to perform its magic. 

Let’s now look at the actual structure of a pass and walk through the code you’ll need to define and register it.

```c++
#include "llvm/IR/PassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
// other includes

using namespace llvm;

// Pass logic
class MyXPass : public PassInfoMixin<MyXPass> {
public:
    PreservedAnalyses run(X &M, XAnalysisManager &AM) {
        // ...
        // Return what analyses are preserved
        return PreservedAnalyses::all();
    }
};

// Registration for `opt`
llvm::PassPluginLibraryInfo getMyXPassPluginInfo() {
    return {LLVM_PLUGIN_API_VERSION, "MyXPass", LLVM_VERSION_STRING,
            [](PassBuilder &PB) {
                PB.registerPipelineParsingCallback(
                    [](StringRef Name, XPassManager &MPM,
                       ArrayRef<PassBuilder::PipelineElement>) {
                        if (Name == "my-pass") {
                            MPM.addPass(MyXPass());
                            return true;
                        }
                        return false;
                    });
            }};
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getMyXPassPluginInfo();
}
```

What we see above is a template, usable for the type of passes we introduced in previous sections. 

>Note that the 'X' in the parameter type of the (member) functions denote Module, Function or Loop.


As mentioned in the [previous article](https://dev.to/cppchedy/compiling-llvm-and-running-your-first-dummy-pass-2an5), we can see two distinct part of the code:
- A class representing the pass logic.
- A function that registers our pass with so it can be recognized by `opt` via `--passes=`.

Let's start over by `opt` registration part.

### Registering the Pass

To run our pass with `opt`, we need to write the following piece of code:

```c++
llvm::PassPluginLibraryInfo getMyXPassPluginInfo() {
    return {LLVM_PLUGIN_API_VERSION, "MyXPass", LLVM_VERSION_STRING,
            [](PassBuilder &PB) {
                PB.registerPipelineParsingCallback(
                    [](StringRef Name, XPassManager &MPM,
                       ArrayRef<PassBuilder::PipelineElement>) {
                        if (Name == "my-pass") {
                            MPM.addPass(MyXPass());
                            return true;
                        }
                        return false;
                    });
            }};
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getMyXPassPluginInfo();
}
```

Let's walk through it step by step. We have a mandatory function, `llvmGetPassPluginInfo`, with external linkage that provides metadata of our pass/plugin to `opt` in order to accept or reject the loading. This function returns a `PassPluginLibraryInfo` object containing things like our pass name, version, LLVM version and a lambda accepting a reference to a `PassBuilder` object. This is the object responsible for registering our pass in the **new** Pass manager system(_you can look up the legacy pass manager if you are interested_).  

Zooming on the "registration" lambda, we see that it just hooks into LLVM pipeline parsing with a callback, triggered to match the input passed to `--passes=` in `opt`. The last bit is the instantiation of our pass, executed if the input matches our pass name.

### Core Function

The pass essence is written in the following listing:

```c++
class MyXPass : public PassInfoMixin<MyXPass> {
public:
    PreservedAnalyses run(X &M, XAnalysisManager &AM) {
        // ...
        // Return what analyses are preserved
        return PreservedAnalyses::all();
    }
};
```

What you need to pay attention to are:
- **Class PassInfoMixin**
- **Run member function** 
- **PreservedAnalyses**
- **XAnalysisManager**
- **X**

We inherit from `PassInfoMixin`. This class provides the boilerplate required by LLVM pass system to integrate with the new pass manger. By inheriting from this mixin, our pass gets recognized and managed by LLVM.

The **run method** is the core of our pass. It gets called when LLVM decides to execute our pass logic. This method has two arguments:
- `X &M`: This represents the IR unit your pass operates on.(e.g Module.)
- `XAnalysisManager &AM` : Gives access to **analysis results** and allow us to query or preserve information other passes might depend on.

The body of `run` member function contains the logic of our pass. At the end of this member function, we indicate to the Pass Manager that the analysis results are still valid. Specifically, if you didn't transform anything you can just return `PreservedAnalyses::all()`,  Which tells the Pass Manager that you didn't make any change on the IR.

## Getting our hands dirty for Real

That’s enough setup! Time to get our hands dirty with some real code. Let’s write a pass that counts all the `add` instructions in a module.

### Counting `Add` instructions Pass

The following is the code of our pass:

```c++
#include "llvm/IR/PassManager.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"

using namespace llvm;

// Pass logic
class AddCounterPass : public PassInfoMixin<AddCounterPass> {
public:
  PreservedAnalyses run(Module &M, ModuleAnalysisManager &) {
    int count = 0;
    for (auto &F : M.functions()) { // iterate over functions
      errs() << "Analyzing function: " << F.getName() << "\n";
      for (auto &BB : F) {// iterate over basic blocks of a function
        for (auto &I : BB) {// iterate over instruction in a BB
          if (I.getOpcode() == Instruction::Add) {
            ++count;
          }
        }
      }
    }
    errs() << "Number of add inst in the module : " << count << "\n";
    return PreservedAnalyses::all();
  }
};

// Registration for `opt`
llvm::PassPluginLibraryInfo getAddCounterPassPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "AddCounterPass", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, ModulePassManager &MPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "add-counter-pass") {
                    MPM.addPass(AddCounterPass());
                    return true;
                  }
                  return false;
                });
          }};
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getAddCounterPassPluginInfo();
}
```

The registration bits are repetitive and I won't go over them here. Let's dig into the pass logic. 

```c++
PreservedAnalyses run(Module &M, ModuleAnalysisManager &) {
    int count = 0;
    for (auto &F : M.functions()) { // iterate over functions
      errs() << "Analyzing function: " << F.getName() << "\n";
      for (auto &BB : F) {// iterate over basic blocks of a function
        for (auto &I : BB) {// iterate over instruction in a BB
          if (I.getOpcode() == Instruction::Add) {
            ++count;
          }
        }
      }
    }
    errs() << "Number of add inst in the module : " << count << "\n";
    return PreservedAnalyses::all();
  }
```

The logic here is very straightforward once you get past the unfamiliar LLVM APIs. We iterate over all the functions in a `Module`, and for each function, we iterate through its basic blocks. Then, for every instruction inside each basic block, we check whether it matches `Instruction::Add`. If it does, we increment our counter. Simple, right?

You can think of a `Module` as representing a translation unit, containing one or more IR functions. A **BasicBlock** is just a sequence of instructions with a single entry point and a single exit (typically a branch or terminator instruction).


Note that LLVM offers multiple ways to match instruction. we may visit them in a future post.

To compile the pass, you can adapt the previous `CMakeLists.txt` file or just issue the following:

```bash
clang++ -fPIC -shared -o AddCounterPass.so add_counter_pass.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core passes` -Wl,-rpath,`llvm-config --libdir`
```

Pay attention to which clang you are invoking. It may cause some trouble if you didn't use the one we built from source.


### Testing the Pass

We will be using the following listing to test our pass:

```c++
int randomAdditionFn(int a, int b, int c) {
  return a + b+ c;
}

int anotherAddFn(int a, int b, int c, int d) {
  int interm1 = a+ c;
  int interm2 = b + d;
  return interm1 + interm2;
}

int mixedAdd(int a, int b) {
  return a * b + 13;
}
```

And the command to turn it to IR:

```bash
clang++ -S -emit-llvm test.cpp -o test.ll
```

Run the pass with:

```bash
opt -load-pass-plugin=./AddCounterPass.so -passes=add-counter-pass -disable-output < test.ll
```

You will get something like this, if you used the same C++ code:

```bash
Analyzing function: _Z16randomAdditionFniii
Analyzing function: _Z12anotherAddFniiii
Analyzing function: _Z8mixedAddii
Number of add inst in the module : 6
```

## Conclusion 

In this post, we covered some theory around LLVM passes and got a taste of how the pass system actually works by writing a simple one ourselves. We intentionally glossed over a few details to keep the focus on getting something running.

In future articles, we’ll circle back to those parts and dig deeper into the pieces we’ve only touched on so far.
