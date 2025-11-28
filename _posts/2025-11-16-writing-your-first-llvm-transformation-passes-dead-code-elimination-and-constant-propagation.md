---
layout: post
title: "Writing your first LLVM Transformation Passes - Dead Code Elimination and Constant Propagation"
categories: [compilers]
tags: [llvm, llvm-pass, how-to, dce, constant-propagation]
---
## Introduction

In the previous [article](https://cppchedy.github.io/blog/compilers/2025/10/21/writing-yout-first-llvm-plugin-pass-counting-add-instructions.html), we learned about LLVM passes and wrote our first analysis pass, which counted the number of `add` instructions inside a module. In this new post, I will explore more of LLVM by implementing transformation passes that make use of several LLVM APIs. This will expose us to different ways of processing the IR and highlight the facilities LLVM provides to its users.

The good news is that this article does not require much theory so we will dive directly into the code. Note that I will not be showing all of the pass code (specifically, the registration boilerplate), but I will provide a Gist link for the sake of completeness.

## Dead Code Elimination 

Dead Code Elimination (DCE) is one of the first optimizations a compiler applies to remove unused results and regions of code that are never reached. For example, expressions that appear after a `return` statement or conditions that always evaluate to false.

To keep things simple, I am redefining DCE for our beginner-level purposes. Our pass will remove only unused instructions inside a basic block and will not handle unreachable code.

Concretely, consider the following IR code:

```llvm-ir
define i32 @multip(i32 %a) {
entry:
  %unused = add i32 %a, %a
  %res = mul i32 %a, %a
  ret i32 %res
}
```
where we take an `i32`, add it to itself, and then multiply the original value (not the result of the addition) by itself before returning the result of that multiplication. When we apply our simplified DCE pass, the resulting code contains only the multiplication instruction, as shown in the following listing:

```llvm-ir
define i32 @multip(i32 %a) {
entry:
  %res = mul i32 %a, %a
  ret i32 %res
}
```
### Our Downsized DCE Pass

As mentioned in the introduction, I will only be showing the pass class and not the registration code. Following what we discussed in the previous subsection, here is what our pass looks like:

```c++
struct DCECustomPass : PassInfoMixin<DCECustomPass> {
  void removeUnusedResultsIfAny(BasicBlock* BB) {
    for (auto it = BB->begin(); it != BB->end();++it) {
        Instruction *It = &*it;
        if(!It->isTerminator() && It->use_empty()) {
          It->eraseFromParent();
        }
    }
  }

  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    errs() << "Analyzing function: " << F.getName() << "\n";
    for (auto &BB : F) {
        removeUnusedResultsIfAny(&BB);
      }
    }
    for (auto &BB : F) {
      errs() << "  BasicBlock:\n";
      for (auto &I : BB) {
        errs() << "    Instruction: " << I << "\n";
      }
    }
    return PreservedAnalyses::none();
  }
};
```

Let’s break it down. First, we note two member functions: our usual `run` and `removeUnusedResultsIfAny`. The `run` method iterates over the basic blocks of a function and delegates the transformation work to `removeUnusedResultsIfAny`. Afterwards, it prints the updated IR and informs the pass system that none of the analyses have been preserved.

The `removeUnusedResultsIfAny` method checks each instruction in the given basic block and removes it if it is unused. It also checks for terminator instructions to avoid removing return statements and similar control-flow terminators.

You can refer to one of the previous articles for instructions on how to compile the pass. Next, we run it with:
```bash
opt -load-pass-plugin=./DCECustomPass.so -passes=dce-pass -disable-output < test.ll
```

and the result is :

```bash
💥💥💥💥
```

Yeah, we segfault marvelously! 😀

This bug is a common one you often see in beginner’s code. It occurs because we remove instructions while iterating over the basic block. When the current instruction is erased, the iterator becomes invalid, and advancing it afterwards can trigger a null-pointer dereference.

Fortunately, the fix is simple: we just need to advance the iterator to the next element before removing the current instruction, as shown below:
```c++
void removeUnusedResultsIfAny(BasicBlock* BB) {
  for (auto it = BB->begin(); it != BB->end();/*removed ++it*/) {
    Instruction *It = &*it;
    ++it;// move to here. we go to the next before we remove
    if(!It->isTerminator() && It->use_empty()) {
      It->eraseFromParent();
    }
  }
}
```
As you can see, when your transformation involves removing instructions, you must ensure that the iterator is advanced to the next instruction before performing any processing on the current one.

Running `opt` with the patched code yields:

```bash
Analyzing function: multip
  BasicBlock:
    Instruction:   %res = mul i32 %a, %a
    Instruction:   ret i32 %res
```

## Constant Propagation Pass

This is an optimization pass that aims to replace computations involving known values with their results at compile time. In other words, if a binary operation uses two hard-coded constants, the compiler can evaluate it immediately and propagate the resulting value to any instructions that depend on it.

For example, consider the following IR listing:
```llvm-ir
define i32 @testConstProp(i32 %a) {
entry:
  %x = add i32 1, 2
  %y = mul i32 %x, 3
  %z = sub i32 %y, 6
  %w = udiv i32 %z, 3
  %res = add i32 %a, %w
  ret i32 %res
}
```

Where we are adding, multiplying, subtracting, and dividing constant operands, and finally adding the parameter `%a` to the result of those computations.

Our constant propagation pass transforms the IR above into:

```llvm-ir
define i32 @testConstProp(i32 %a) {
entry:
  %res = add i32 %a, 1
  ret i32 %res
}
```

The pass removes instructions whose results can be computed at compile time and propagates the corresponding constant values. This process continues until it reaches an `add` instruction where one of the operands is a variable.

### Constant Propagation Pass for `add` 

Let’s start coding! The following listing shows the `run` member function of our pass, which performs constant propagation for both `add` and `fadd` instructions:

```c++
PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    errs() << "Analyzing function: " << F.getName() << "\n";
    LLVMContext &context = F.getContext();
    for (auto &BB : F) {
      for (auto it = BB.begin(); it != BB.end();) {
        if (auto *BO = dyn_cast<BinaryOperator>(&*it++)) {
          ConstantInt *lOprd = nullptr;
          ConstantInt *rOprd = nullptr;
          ConstantFP *flOprd = nullptr;
          ConstantFP *frOprd = nullptr;

          if (PatternMatch::match(BO,
                                  m_Add(PatternMatch::m_ConstantInt(lOprd),
                                        PatternMatch::m_ConstantInt(rOprd)))) {
            auto newVal = lOprd->getValue() + rOprd->getValue();

            ConstantInt *constInt = ConstantInt::get(context, newVal);
            BO->replaceAllUsesWith(constInt);
            BO->eraseFromParent();
          } else if (PatternMatch::match(
                         BO, m_FAdd(PatternMatch::m_ConstantFP(flOprd),
                                    PatternMatch::m_ConstantFP(frOprd)))) {
            auto newVal = flOprd->getValue() + frOprd->getValue();

            ConstantFP *constFP = ConstantFP::get(context, newVal);
            BO->replaceAllUsesWith(constFP);
            BO->eraseFromParent();
          } else {
            errs() << "Instruction not supported yet" << *BO << "\n";
          }
        }
      }
    }
    for (auto &BB : F) {
      errs() << "  BasicBlock:\n";
      for (auto &I : BB) {
        errs() << "    Instruction: " << I << "\n";
      }
    }
    return PreservedAnalyses::none();
  }		
```

Let’s walk through the code step by step. The pass iterates over every instruction in the current function and tries to match them with `add` and `fadd` instructions that have constant operands. Note the use of LLVM’s pattern matching facility. Previously, we relied directly on instruction opcodes, but pattern matching is more powerful here because it extracts the operands automatically when a match is found.

When a match occurs, we perform the addition, replace all of the instruction’s uses in the rest of the code, and then remove the original instruction. Each iteration starts by filtering for binary operations to ensure only relevant instructions are processed.

Finally, we print the transformed IR to observe the effect of the constant propagation.

## Used LLVM facilities

One of the goals of this post was to explore and demonstrate more of the facilities provided by LLVM for writing passes. The following table summarizes nearly all of the APIs used throughout the previous sections:

| API (symbol)                                                |                                      Kind | What it does (in this pass/code)                                                                                                                 | Header           |
| ----------------------------------------------------------- | ----------------------------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| `PassInfoMixin<T>`                                          |                 CRTP helper class / mixin | Base mixin used when writing the new-style (PM) pass type; gives pass-related helpers and boilerplate.                                           | `#include "llvm/IR/PassManager.h"`      |
| `PreservedAnalyses`                                         |                       Class / return type | Represents which analyses are preserved after a pass; `PreservedAnalyses::none()` signals nothing preserved.                                     | `#include "llvm/IR/PassManager.h"`      |
| `FunctionAnalysisManager`                                   |                  Class (analysis manager) | Manager for function analyses (passed into `run`); used by the new pass manager interface.                                                       | `#include "llvm/IR/PassManager.h"`      |
| `Function`                                                  |                      LLVM IR type / class | Represents a function; the pass iterates over `Function &F` to visit basic blocks and instructions.                                              | `#include "llvm/IR/Function.h"`         |
| `BasicBlock`                                                |                      LLVM IR type / class | Represents a basic block; used to iterate instructions and perform transformations per-block.                                                    | `#include "llvm/IR/BasicBlock.h"`       |
| `Instruction`                                               |                             LLVM IR class | Base class for all instructions; methods like `isTerminator()`, `use_empty()`, `eraseFromParent()` are used.                                     | `#include "llvm/IR/Instruction.h"`      |
| `Instruction::isTerminator()`                               |                                    Method | Returns true if the instruction is a terminator (e.g., `ret`, `br`) — used to avoid removing terminators.                                        | `llvm/IR/Instruction.h`                 |
| `Instruction::use_empty()`                                  |                                    Method | Checks whether the instruction has zero uses (no other IR references its result). Key to DCE.                                                    | `llvm/IR/Instruction.h`                 |
| `Instruction::eraseFromParent()`                            |                                    Method | Removes (deletes) the instruction from its parent basic block and frees it.                                                                      | `llvm/IR/Instruction.h`                 |
| `errs()`                                                    |           Function / global `raw_ostream` | LLVM’s error/diagnostic output stream (like `std::cerr`); used to print debugging/transformed IR.                                                | `#include "llvm/Support/raw_ostream.h"` |
| `Function::getName()`                                       |                                    Method | Returns the function name (StringRef) — used when printing which function is being analyzed.                                                     | `llvm/IR/Function.h`                    |
| `Function::getContext()`                                    |                                    Method | Returns the `LLVMContext &` associated with the function; needed to create `Constant*`s.                                                         | `llvm/IR/Function.h`                    |
| `LLVMContext`                                               |                                     Class | Context object holding uniqued types, constants, metadata, etc.; needed when creating constants.                                                 | `#include "llvm/IR/LLVMContext.h"`      |
| `BasicBlock::begin()` / `BasicBlock::end()`                 |                       Methods / iterators | Provide iterator access to a block’s instructions (`Instruction` iterators) for manual iteration.                                                | `llvm/IR/BasicBlock.h`                  |
| `dyn_cast<T>(Value*)`                                       |                       LLVM RTTI-like cast | Safe downcast that returns `T*` if the value is of type `T` (otherwise nullptr). Used to detect `BinaryOperator`.                                | `#include "llvm/Support/Casting.h"`     |
| `BinaryOperator`                                            | LLVM IR class (subclass of `Instruction`) | Represents binary operators (`add`, `mul`, `sub`, `udiv`, `fadd`, ...); used to detect/operate on binary arithmetic instructions.                | `#include "llvm/IR/Instructions.h"`     |
| `ConstantInt`                                               |                                     Class | Represents integer constants; used to inspect integer operands and to create folded integer `ConstantInt`s.                                      | `#include "llvm/IR/Constants.h"`        |
| `ConstantFP`                                                |                                     Class | Represents floating-point constants; used for `fadd` constant folding.                                                                           | `#include "llvm/IR/Constants.h"`        |
| `PatternMatch::match(...)`                                  |                   Pattern matching helper | High-level matcher to test instruction shapes (e.g., `m_Add(m_ConstantInt(...), m_ConstantInt(...))`) and extract operands.                      | `#include "llvm/IR/PatternMatch.h"`     |
| `m_Add`, `m_FAdd` (pattern matchers)                        |        Pattern matcher macros / functions | Match `add` and `fadd` binary operations in `PatternMatch::match`.                                                                               | `llvm/IR/PatternMatch.h`                |
| `PatternMatch::m_ConstantInt`, `PatternMatch::m_ConstantFP` |                                  Matchers | Match constant integer / floating operands and bind them to variables on success.                                                                | `llvm/IR/PatternMatch.h`                |
| `APInt` (implicitly via `ConstantInt::getValue()`)          |         Arbitrary-precision integer class | Holds the integer value extracted from `ConstantInt` (`getValue()` returns `APInt`), used for arithmetic on constants.                           | `#include "llvm/ADT/APInt.h"`           |
| `APFloat` (implicitly via `ConstantFP::getValue()`)         |                Floating-point value class | Holds floating-point data returned by `ConstantFP::getValue()` (used for constant fp arithmetic).                                                | `#include "llvm/IR/APFloat.h"`          |
| `ConstantInt::get(context, APInt)`                          |                            Static factory | Creates a `ConstantInt` from an `APInt` and `LLVMContext`. Used to replace folded integer expressions.                                           | `llvm/IR/Constants.h`                   |
| `ConstantFP::get(context, APFloat)`                         |                            Static factory | Creates a `ConstantFP` from an `APFloat` and `LLVMContext`.                                                                                      | `llvm/IR/Constants.h`                   |
| `ConstantInt::getValue()`                                   |                                    Method | Returns the `APInt` value of the `ConstantInt`.                                                                                                  | `llvm/IR/Constants.h`                   |
| `ConstantFP::getValue()`                                    |                                    Method | Returns the `APFloat` value of the `ConstantFP`.                                                                                                 | `llvm/IR/Constants.h`                   |
| `Value::replaceAllUsesWith(Value *V)`                       |                       Method (on `Value`) | Replace every use of this `Value` (instruction/result) with another `Value`. Used before erasing the original instruction.                       | `#include "llvm/IR/Value.h"`            |
| `eraseFromParent()` vs `removeFromParent()` distinction     |                             Method nuance | `eraseFromParent()` deletes the instruction and removes it from the block; `removeFromParent()` just unlinks—your code uses `eraseFromParent()`. | `llvm/IR/Instruction.h`                 |
| `StringRef` (implied by `getName()`)                        |             Lightweight string view class | `Function::getName()` returns a `StringRef` representing the function name; used with `errs() <<`.                                               | `#include "llvm/ADT/StringRef.h"`       |


I think pattern matching is one of the most useful features of LLVM, as it greatly simplifies the code.

## Conclusion

We have written two transformation passes:
- Dead Code Elimination (DCE)
- Constant Propagation

We also saw the key points to pay attention to and how LLVM provides facilities that make writing passes easier.
