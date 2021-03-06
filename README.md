![AssemblyScript](https://raw.githubusercontent.com/AssemblyScript/assemblyscript/master/logo.png)
==============

[AssemblyScript](https://github.com/AssemblyScript) defines a subset of [TypeScript](http://www.typescriptlang.org) that it compiles to [WebAssembly](http://webassembly.org). It aims to provide everyone with an existing background in TypeScript and standard JavaScript-APIs with a comfortable way to compile to WebAssembly, eliminating the need to switch between languages or to learn new ones just for this purpose.

Try it out [in your browser](http://assemblyscript.org/try)!

[![npm](https://img.shields.io/npm/v/assemblyscript.svg)](https://www.npmjs.com/package/assemblyscript) [![Build Status](https://travis-ci.org/AssemblyScript/assemblyscript.svg?branch=master)](https://travis-ci.org/AssemblyScript/assemblyscript) [![npm](https://img.shields.io/npm/dm/assemblyscript.svg)](https://www.npmjs.com/package/assemblyscript) <a href="https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=dcode%40dcode.io&item_name=Open%20Source%20Software%20Donation&item_number=dcodeIO%2Fprotobuf.js"><img alt="donate ❤" src="https://img.shields.io/badge/donate-❤-ff2244.svg"></a>

Contents
--------

* [How it works](#how-it-works)<br />
  A few insights to get an initial idea.

* [What to expect](#what-to-expect)<br />
  General remarks on design decisions and trade-offs.

* [Example](#example)<br />
  Basic examples to get you started.

* [Usage](#usage)<br />
  An introduction to the environment and its provided functionality.

* [Command line](#command-line)<br />
  How to use the command line utility.

* [API](#api)<br />
  How to use the API programmatically.

* [Additional documentation](#additional-documentation)<br />
  A list of available documentation resources.

* [Building](#building)<br />
  How to build the compiler and its components yourself.

How it works
------------

Under the hood, AssemblyScript rewires TypeScript's [compiler API](https://github.com/Microsoft/TypeScript-wiki/blob/master/Using-the-Compiler-API.md) to [Binaryen](https://github.com/WebAssembly/binaryen)'s compiler backend. The compiler itself is written in (and based upon) TypeScript and no binary dependencies are required to get started.

Every AssemblyScript program is valid TypeScript syntactically, but not necessarily semantically. The definitions required to start developing in AssemblyScript are provided by [assembly.d.ts](./assembly.d.ts). See also: [Usage](#usage)

The compiler is able to produce WebAssembly binaries (.wasm) as well as their corresponding text format. Both Binaryen's s-expression format (.wast) and, with a little help of [WABT](https://github.com/WebAssembly/wabt), official linear text format (.wat) are supported. See also: [CLI](#command-line)

What to expect
--------------

<details><p>
The most prominent difference of JavaScript and any strictly typed language is that, in JavaScript, a variable can reference a value of any type. This implies that a JavaScript execution environment has to emit additional runtime checks whenever a variable is accessed. Modern JavaScript VMs shortcut the overhead introduced by this and similar dynamic features by generating case-specific code based on statistical information collected just in time (JIT), speeding up execution significantly. Similarily, developers shortcut the overhead of remembering each variable's type by using TypeScript. The combination of both also makes for a good match because it potentially aids the JIT compiler.

Because it has the ability to fall back to dynamic JavaScript features, TypeScript isn't a *strictly* typed language after all. For example, TypeScript supports omittable (i.e. `someParameter?: number`) function parameters resulting in a union type `number | undefined` at runtime, just like it also allows declaring union types explicitly. These constructs are incompatible with a strict, ahead of time (AOT) compiled type system unless additional runtime checks are emitted that'd usually execute slower than similar code running in a VM that has the ability to perform optimizations at runtime. Hence...
</p><summary><strong>TL;DR</strong></summary></details>

Instead of reimplementing TypeScript *as closely as possible* at the expense of performance, AssemblyScript tries to support its features *as closely as reasonable* while not supporting certain dynamic constructs intentionally:

* All types must be annotated to avoid possibly unwanted implicit type conversions
* Optional function parameters require an initializer expression
* Union types (except `classType | null` representing a nullable), `any` and `undefined` are not supported by design
* The result of logical `&&` / `||` expressions is always `bool`

Also note that AssemblyScript is a rather new and ambitious project developed by one guy and a hand full of occasional contributors. Expect bugs and breaking changes. Prepare to fix stuff yourself and to send a PR for it, unless you like the idea enough to consider sponsoring development.

Example
-------

```ts
export function add(a: int, b: double): short {
  return (a + (b as int)) as short;
}
```

Compiles to:

```s
(module
 (type $iFi (func (param i32 f64) (result i32)))
 (memory $0 256)
 (export "memory" (memory $0))
 (export "add" (func $add))
 (func $add (type $iFi) (param $0 i32) (param $1 f64) (result i32)
  (return
   (i32.shr_s
    (i32.shl
     (i32.add
      (get_local $0)
      (i32.trunc_s/f64
       (get_local $1)
      )
     )
     (i32.const 16)
    )
    (i32.const 16)
   )
  )
 )
)
```

See [the examples repository](https://github.com/AssemblyScript/examples) for more.

### Running a module

The stand-alone [loader component](https://github.com/AssemblyScript/loader) provides an easy way to run and work with compiled WebAssembly modules:

```
$> npm install assemblyscript-loader
```

```ts
import load from "assemblyscript-loader"; // JS: var load = require("assemblyscript-loader").load;

load("path/to/module.wasm", {
  imports: {
    ...
  }
}).then(module => {
  ...
  // i.e. call module.exports.main()
});
```

Usage
-----

```
$> npm install assemblyscript --save-dev
```

The environment is configured by either referencing [assembly.d.ts](./assembly.d.ts) directly or by using a `tsconfig.json` that simply extends [tsconfig.assembly.json](https://github.com/AssemblyScript/assemblyscript/blob/master/tsconfig.assembly.json), like so:

```json
{
  "extends": "./node_modules/assemblyscript/tsconfig.assembly.json",
  "include": [
    "./*.ts"
  ]
}
```

The `tsconfig.json`-approach is recommended to inherit other important settings as well.

Once configured, the following AssemblyScript-specific types become available:

Type      | Alias     | Native type | sizeof | Description
----------|-----------|-------------|--------|-------------
`sbyte`   | `int8`    | i32         | 1      | An 8-bit signed integer.
`byte`    | `uint8`   | i32         | 1      | An 8-bit unsigned integer.
`short`   | `int16`   | i32         | 2      | A 16-bit signed integer.
`ushort`  | `uint16`  | i32         | 2      | A 16-bit unsigned integer.
`int`     | `int32`   | i32         | 4      | A 32-bit signed integer.
`uint`    | `uint32`  | i32         | 4      | A 32-bit unsigned integer.
`long`    | `int64`   | i64         | 8      | A 64-bit signed integer.
`ulong`   | `uint64`  | i64         | 8      | A 64-bit unsigned integer.
`uintptr` | -         | i32 / i64   | 4 / 8  | A 32-bit unsigned integer when targeting 32-bit WebAssembly.<br />A 64-bit unsigned integer when targeting 64-bit WebAssembly.
`float`   | `float32` | f32         | 4      | A 32-bit float.
`double`  | `float64` | f64         | 8      | A 64-bit float.
`bool`    | -         | i32         | 1      | A 1-bit unsigned integer.
`void`    | -         | none        | -      | No return type

While generating a warning to avoid type confusion, the JavaScript types `number` and `boolean` resolve to `double` and `bool` respectively.

WebAssembly-specific operations are available as built-in functions that translate to the respective opcode directly:

* **rotl**(value: `int`, shift: `int`): `int`<br />
  Performs the sign-agnostic rotate left operation on a 32-bit integer.
* **rotll**(value: `long`, shift: `long`): `long`<br />
  Performs the sign-agnostic rotate left operation on a 64-bit integer.
* **rotr**(value: `int`, shift: `int`): `int`<br />
  Performs the sign-agnostic rotate right operation on a 32-bit integer.
* **rotrl**(value: `long`, shift: `long`): `long`<br />
  Performs the sign-agnostic rotate right operation on a 64-bit integer.
* **clz**(value: `int`): `int`<br />
  Performs the sign-agnostic count leading zero bits operation on a 32-bit integer. All zero bits are considered leading if the value is zero.
* **clzl**(value: `long`): `long`<br />
  Performs the sign-agnostic count leading zero bits operation on a 64-bit integer. All zero bits are considered leading if the value is zero.
* **ctz**(value: `int`): `int`<br />
  Performs the sign-agnostic count tailing zero bits operation on a 32-bit integer. All zero bits are considered trailing if the value is zero.
* **ctzl**(value: `long`): `long`<br />
  Performs the sign-agnostic count trailing zero bits operation on a 64-bit integer. All zero bits are considered trailing if the value is zero.
* **popcnt**(value: `int`): `int`<br />
  Performs the sign-agnostic count number of one bits operation on a 32-bit integer.
* **popcntl**(value: `long`): `long`<br />
  Performs the sign-agnostic count number of one bits operation on a 64-bit integer.
* **abs**(value: `double`): `double`<br />
  Computes the absolute value of a 64-bit float.
* **absf**(value: `float`): `float`<br />
  Computes the absolute value of a 32-bit float.
* **ceil**(value: `double`): `double`<br />
  Performs the ceiling operation on a 64-bit float.
* **ceilf**(value: `float`): `float`<br />
  Performs the ceiling operation on a 32-bit float.
* **floor**(value: `double`): `double`<br />
  Performs the floor operation on a 64-bit float.
* **floorf**(value: `float`): `float`<br />
  Performs the floor operation on a 32-bit float.
* **sqrt**(value: `double`): `double`<br />
  Calculates the square root of a 64-bit float.
* **sqrtf**(value: `float`): `float`<br />
  Calculates the square root of a 32-bit float.
* **trunc**(value: `double`): `double`<br />
  Rounds to the nearest integer towards zero of a 64-bit float.
* **truncf**(value: `float`): `float`<br />
  Rounds to the nearest integer towards zero of a 32-bit float.
* **nearest**(value: `double`): `double`<br />
  Rounds to the nearest integer tied to even of a 64-bit float.
* **nearestf**(value: `float`): `float`<br />
  Rounds to the nearest integer tied to even of a 32-bit float.
* **min**(left: `double`, right: `double`): `double`<br />
  Determines the minimum of two 64-bit floats. If either operand is `NaN`, returns `NaN`.
* **minf**(left: `float`, right: `float`): `float`<br />
  Determines the minimum of two 32-bit floats. If either operand is `NaN`, returns `NaN`.
* **max**(left: `double`, right: `double`): `double`<br />
  Determines the maximum of two 64-bit floats. If either operand is `NaN`, returns `NaN`.
* **maxf**(left: `float`, right: `float`): `float`<br />
  Determines the maximum of two 32-bit floats. If either operand is `NaN`, returns `NaN`.
* **copysign**(x: `double`, y: `double`): `double`<br />
  Composes a 64-bit float from the magnitude of `x` and the sign of `y`.
* **copysignf**(x: `float`, y: `float`): `float`<br />
  Composes a 32-bit float from the magnitude of `x` and the sign of `y`.
* **reinterpreti**(value: `float`): `int`<br />
  Reinterprets the bits of a 32-bit float as a 32-bit integer.
* **reinterpretl**(value: `double`): `long`<br />
  Reinterprets the bits of a 64-bit float as a 64-bit integer.
* **reinterpretf**(value: `int`): `float`<br />
  Reinterprets the bits of a 32-bit integer as a 32-bit float.
* **reinterpretd**(value: `long`): `double`<br />
  Reinterprets the bits of a 64-bit integer as a 64-bit double.
* **current_memory**(): `int`<br />
  Returns the current memory size in units of pages. One page is 64kb.
* **grow_memory**(value: `uint`): `int`<br />
  Grows linear memory by a given unsigned delta of pages. One page is 64kb. Returns the previous memory size in units of pages or `-1` on failure.
* **unreachable**(): `void`<br />
  Emits an unreachable operation that results in a runtime error when executed.
* **load**<`T`>(offset: `uintptr`): `T`<br />
  Loads a value of the specified type from memory.
* **store**<`T`>(offset: `uintptr`, value: `T`): `void`<br />
  Stores a value of the specified type to memory.

The following AssemblyScript-specific operations are implemented as built-ins as well:

* **sizeof**<`T`>(): `uintptr`<br />
  Determines the byte size of the specified core or class type. Compiles to a constant.
* **unsafe_cast**<`T1`,`T2`>(value: `T1`): `T2`<br />
  Casts a value of type `T1` to a value of type `T2`. Useful for casting classes to pointers and vice-versa. Does not perform any checks.
* **isNaN**(value: `double`): `bool`<br />
  Tests if a 64-bit float is a NaN.
* **isNaNf**(value: `float`): `bool`<br />
  Tests if a 32-bit float is a NaN.
* **isFinite**(value: `double`): `bool`<br />
  Tests if a 64-bit float is finite.
* **isFinitef**(value: `float`): `bool`<br />
  Tests if a 32-bit float is finite.

These constants are present as immutable globals (note that optimizers might inline them):

* **NaN**: `double`<br />
  NaN (not a number) as a 64-bit float.
* **NaNf**: `float`<br />
  NaN (not a number) as a 32-bit float.
* **Infinity**: `double`<br />
  Positive infinity as a 64-bit float.
* **Infinityf**: `float`<br />
  Positive infinity as a 32-bit float.

By default, [AssemblyScript's memory management runtime](https://github.com/AssemblyScript/runtime) will be linked statically:

* **memcpy**(dest: `uintptr`, src: `uintptr`, size: `uintptr`): `uintptr`<br />
  Copies data from one chunk of memory to another.
* **memset**(dest: `uintptr`, c: `int`, size: `uintptr`): `uintptr`<br />
  Sets a chunk of memory to the provided value `c`. Usually used to reset it to all `0`s.
* **memcmp**(vl: `uintptr`, vr: `uintptr`, n: `uintptr`): `int`<br />
  Compares a chunk of memory to another. Returns `0` if both are equal, otherwise `vl[i] - vr[i]` at the first difference's byte offset `i`.
* **malloc**(size: `uintptr`): `uintptr`<br />
  Allocates a chunk of memory of the specified size.
* **realloc**(ptr: `uintptr`, size: `uintptr`): `uintptr`<br />
  Changes the size of an allocated memory block.
* **free**(ptr: `uintptr`): `void`<br />
  Frees a previously allocated chunk of memory.

Linking in the runtime adds up to 14kb to a module, but the optimizer is able to eliminate unused runtime code. Once WebAssembly exposes the garbage collector natively, there'll be other options as well. If the runtime has been excluded through `--noRuntime`, its methods will be imported where referenced (i.e. when using `new`). Also note that manually calling `grow_memory` where the runtime is present will most likely break it.

Type coercion requires an explicit cast where precision or signage is lost respectively is implicit where it is maintained. For example, to cast a `double` to an `int`:

```ts
function example(value: double): int {
  return value as int; // translates to the respective opcode
}
```

Global WebAssembly imports can be `declare`d anywhere while WebAssembly exports are `export`ed from the entry file (the file specified when calling `asc` or `Compiler.compileFile`). Aside from that, imports and exports work just like in TypeScript.

```ts
// entry.ts

import { myOtherExportThatDoesntBecomeAWebAssemblyExport } from "./imported";

declare function myImport(): void;

export function myExport(): void {
  myOtherExportThatDoesntBecomeAWebAssemblyExport();
}
```

Currently, imports can also be pulled from different namespaces by separating the namespace and the function with a `$` character.

```ts
declare function Math$random(): double;
```

Command line
------------

The command line compiler `asc` works similar to TypeScript's `tsc`:

```
Syntax: asc [options] entryFile

Options:

 --config, -c       Specifies a JSON configuration file with command line options.
                    Will look for 'asconfig.json' in the entry's directory if omitted.

 --outFile, -o      Specifies the output file name. Emits text format if ending with .wast
                    (sexpr) or .wat (linear). Prints to stdout if omitted.

 --optimize, -O     Runs optimizing binaryen IR passes.

 --validate, -v     Validates the module.

 --quiet, -q        Runs in quiet mode, not printing anything to console.

 --target, -t       Specifies the target architecture:

                    wasm32  Compiles to 32-bit WebAssembly [default]
                    wasm64  Compiles to 64-bit WebAssembly

 --textFormat, -f   Specifies the format to use for text output:

                    sexpr   Emits s-expression syntax (.wast) [default]
                    linear  Emits official linear syntax (.wat)

                    Text format only is emitted when used without --textFile.

 --textFile         Can be used to save text format alongside a binary in one command.

 --noTreeShaking    Whether to disable built-in tree-shaking.

 --noImplicitConversion  Whether to disallow implicit type conversions.

 --noRuntime        Whether to exclude the runtime.

 --exportRuntime, -e  Runtime functions to export, defaults to 'malloc' and 'free'. [multiple]

 --help, -h         Displays this help message.
```

A configuration file (usually named `asconfig.json`) using the long option keys above plus a special key `entryFile` specifying the path to the entry file can be used to reuse options between invocations.

API
---

It's also possible to use the API programmatically:

* **Compiler.compileFile**(filename: `string`, options?: `CompilerOptions`): `binaryen.Module | null`<br />
  Compiles the specified entry file to a WebAssembly module. Returns `null` on failure.

* **Compiler.compileString**(source: `string`, options?: `CompilerOptions`): `binaryen.Module | null`<br />
  Compiles the specified entry file source to a WebAssembly module. Returns `null` on failure.

* **Compiler.lastDiagnostics**: `typescript.Diagnostic[]`<br />
  Contains the diagnostics generated by the last invocation of `compilerFile` or `compileString`.

* **CompilerOptions**<br />
  AssemblyScript compiler options.

  * **silent**: `boolean`<br />
    Whether compilation shall be performed in silent mode without writing to console. Defaults to `false`.
  * **target**: `CompilerTarget | string`<br />
    Specifies the target architecture. Defaults to `CompilerTarget.WASM32`.
  * **noTreeShaking**: `boolean`<br />
    Whether to disable built-in tree-shaking. Defaults to `false`.
  * **noImplicitConversion**: `boolean`<br />
    Whether to disallow implicit type conversions. Defaults to `false`.
  * **noRuntime**: `boolean`<br />
    Whether to exclude the runtime.
  * **exportRuntime**: `string[]`<br />
    Runtime functions to export, defaults to 'malloc' and 'free'.

 * **CompilerTarget**<br />
   Compiler target.

   * **WASM32**<br />
     32-bit WebAssembly target using uint pointers.
   * **WASM64**<br />
     64-bit WebAssembly target using ulong pointers.

### Example

```ts
import { Compiler, CompilerTarget, CompilerMemoryModel, typescript } from "assemblyscript";

const module = Compiler.compileString(`
export function add(a: int, b: int): int {
  return a + b;
}
`, {
  target: CompilerTarget.WASM32,
  silent: true
});

console.error(typescript.formatDiagnostics(Compiler.lastDiagnostics));
if (!module)
  throw Error("compilation failed");

module.optimize();

if (!module.validate())
  throw Error("validation failed");

const textFile = module.emitText();
const wasmFile = module.emitBinary();

...

module.dispose();
```

Remember to call `binaryen.Module#dispose()` once you are done with a module to free its resources. This is necessary because binaryen.js has been compiled from C/C++ and doesn't provide automatic garbage collection.

Additional documentation
------------------------

#### AssemblyScript

* [Standard Library Documentation](http://dcode.io/AssemblyScript/std)
* [API Documentation](http://dcode.io/AssemblyScript/api)

#### WebAssembly

* [WebAssembly Design Documents](https://github.com/WebAssembly/design)

Building
--------

Clone the GitHub repository including submodules and install the development dependencies:

```
$> git clone --recursive https://github.com/AssemblyScript/assemblyscript.git
$> cd assemblyscript
$> npm install
```

Afterwards, to build the distribution files to *dist/*, run:

```
$> npm run build
```

**Note** that the first invocation of `build` also builds the TypeScript submodule (lib/typescript) and may take some time.

To run the [tests](./tests) (ideally on node.js >= 8):

```
$> npm test
```

To build the documentation to the [website repostory](https://github.com/AssemblyScript/website) checked out next to this repository, run:

```
$> npm run docs
```

---

License: [Apache License, Version 2.0](https://opensource.org/licenses/Apache-2.0)
