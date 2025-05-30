# Deferred re-exports

## Status

Champion(s): _Nicolò Ribaudo_

Author(s): _Nicolò Ribaudo_

Stage: 2

## Introduction

Web applications often have a large amount of JavaScript code, which can have a significant impact on startup time. One way to reduce this impact is by carefully loading as little code needed at any time, potentially prefetching code that is likely to be needed in the future. However, this has proven to be difficult in practice, often leading to under-optimized web applications.

The [`import defer` proposal](https://github.com/tc39/proposal-defer-import-eval/) tackled part of this problem, by allowing with minimal friction to defer execution of code that is not needed during the application startup. It does so by introducing syntax to mark _import_ declarations as "deferrable until when their exports are accessed":

```js
// This declaration _loads_ helpers.js and its dependencies,
// without executing it yet.
import defer * as helpers from "./helpers.js";

function fn() {
  // helpers.js is only executed at this point.
  helpers.doSomething();
}
```

As part of the proposal, we considered ([2023-11 slides](https://docs.google.com/presentation/d/1l-H2ntEDZGAWvtuOup1TJdylZsV1epKVSejVM-GwHLU), [2024-04 `export ... from` slides](https://docs.google.com/presentation/d/1iM5cRgdRXLWLq_GxgRvzYmUTXEK6gzH_8QNgLKMmv7o)) adding similar functionality to `export ... from` declaration. However, `export ... from` declaration have the opportunity to also skip _loading_ of unused modules, and we thus when advancing `import defer` to Stage 2.7 we decided ([2024-04 `import defer` slides](https://docs.google.com/presentation/d/1oPEF8nA9Iq5cAqjN-FqMigNNfz6lWCUbNfIsEjRXf4Y)) to keep the equivalent `export ... from` feature behind as a separate proposal.

## Problem statement

A common pattern that libraries adopt to improve DX of their consumers is to have a single entry point that re-exports all the public APIs. This however has a problem: it leads to a lot of unused code being included in the module graph.

Some tools workaround this problem using a technique called "tree-shaking": they trace the dependencies of the individual bindings exported by a module, and remove the transitive imports (or re-exports) that are only needed for unused exports. However, this operation is not always possible due to side effects that may caused by modules being loaded: different tools choose different trade-offs between code size and correctness, often leading to under-optimization of web applications or to difficult-to-debug issues caused by non-pure modules not being executed as expected. This technique is also _not_ possible when running ESM directly in the browser, as it requires whole-program analysis.

## Proposal

This proposal aims to address the problem by allowing libraries to mark re-exports as "ignore if unused", defining:
1. clear semantics to follow, rather than relying on tooling-defined heuristics;
2. semantics that can also be implemented by native JS platforms to avoid loading unused code;
3. an integration with the `import defer` proposal, to allow these re-exports to benefit from the same "after loading, only execute if actually needed" semantics.

This proposal will use the below syntax, parallel to the [import defer](https://github.com/tc39/proposal-defer-import-eval/) proposal:
```js
// math.js

export defer { add } from "./math/add.js";
export defer { sub } from "./math/sub.js";
```

When a module imports the module above using, for example, `import { add } from "./math.js";`, it will only _load_ (and thus, _execute_) `./math.js` and `./math/add.js`, skipping `./math/sub.js` and all its dependencies.

### Execution order

Deferred re-exports are executed _after_ (if they are executed) the module that re-exports them, in the order they are re-exported. Given this example:
```js
// barrel.js
export defer { a } from "./a.js";
export { b } from "./b.js";
export defer { c } from "./c.js";
export { d } from "./d.js";
export defer { e } from "./e.js";
```
```js
// entrypoint.js
import { e, a, d } from "./barrel.js";
```
The execution order will be `b.js`, `d.js`, `barrel.js`, `a.js`, `e.js`, `entrypoint.js`.

Always executing `defer-exported` modules after the module that re-exports them allows more consistency across different types of module graphs, compared to an execute-what-needed-in-source-order.

<details>
  <summary>Comparison examples</summary>
  All examples import the `./barrel.js` file as defined above.

  <table>
  <thead>
  <tr><th>Title</th><th>Modules</th><th>Executing <code>export defer</code> at the end</th><th>Executing <code>export defer</code> interleaved</th></tr>
  </thead>
  <tbody>
  <tr>
  <td>No deferred bindings imported</td>
  <td>

  ```javascript
  // entry.js
  import { b } from "./barrel.js"
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./entry.js`
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./entry.js`
  </td>
  </tr>
  <tr>
  <td>First deferred binding imported</td>
  <td>

  ```javascript
  // entry.js
  import { a } from "./barrel.js"
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./a.js`
  - `./entry.js`
  </td>
  <td>

  - `./a.js`
  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./entry.js`
  </td>
  </tr>
  <tr>
  <td>Middle deferred binding imported</td>
  <td>

  ```javascript
  // entry.js
  import { c } from "./barrel.js"
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./c.js`
  - `./entry.js`
  </td>
  <td>

  - `./b.js`
  - `./c.js`
  - `./d.js`
  - `./barrel.js`
  - `./entry.js`
  </td>
  </tr>
  <tr>
  <td>Two deferred bindings imported</td>
  <td>

  ```javascript
  // entry.js
  import { a, c } from "./barrel.js"
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./a.js`
  - `./c.js`
  - `./entry.js`
  </td>
  <td>

  - `./a.js`
  - `./b.js`
  - `./c.js`
  - `./d.js`
  - `./barrel.js`
  - `./entry.js`
  </td>
  </tr>
  <tr>
  <td>Two deferred bindings imported by two different files</td>
  <td>

  ```javascript
  // entry.js
  import "./sub-1.js";
  import "./sub-2.js";
  ```

  ```javascript
  // sub-1.js
  import { a } from "./barrel.js";
  ```

  ```javascript
  // sub-2.js
  import { c } from "./barrel.js";
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./a.js`
  - `./sub-1.js`
  - `./c.js`
  - `./sub-2.js`
  - `./entry.js`
  </td>
  <td>

  - `./a.js`
  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./sub-1.js`
  - `./c.js`
  - `./sub-2.js`
  - `./entry.js`
  </td>
  </tr>
  <tr>
  <td>Two deferred bindings imported by two different files (opposite order)</td>
  <td>

  ```javascript
  // entry.js
  import "./sub-2.js";
  import "./sub-1.js";
  ```

  ```javascript
  // sub-2.js
  import { c } from "./barrel.js";
  ```

  ```javascript
  // sub-1.js
  import { a } from "./barrel.js";
  ```
  </td>
  <td>

  - `./b.js`
  - `./d.js`
  - `./barrel.js`
  - `./c.js`
  - `./sub-2.js`
  - `./a.js`
  - `./sub-1.js`
  - `./entry.js`
  </td>
  <td>

  - `./b.js`
  - `./c.js`
  - `./d.js`
  - `./barrel.js`
  - `./sub-2.js`
  - `./a.js`
  - `./sub-1.js`
  - `./entry.js`
  </td>
  </tr>
  </tbody>
  </table>
</details>

The proposed ordering is also much simpler to polyfill in tools, which can extract the `export defer` declaration from imported files and replace them with `import` statements in the importer module.

### Integration with `import defer`

The `import defer` proposal established that the `defer` keyword means "only execute this module when I actually need it", when using _namespace_ imports. On module namespace objects, `export defer` would follow similar semantics:
```js
// math.js
export defer { add } from "./math/add.js";
export defer { sub } from "./math/sub.js";
export { mul } from "./math/mul.js";
```
```js
// index.js
import * as math from "./math.js"; // Loads everything, executes ./math.js and ./math/mul.js

math.add; // Executes ./math/add.js
math.sub; // Executes ./math/sub.js
```

This allows even more control when pairing `export defer` with `import defer`:
```js
// math2.js
export defer { add } from "./math/add.js";
export defer { sub } from "./math/sub.js";
```
```js
// index2.js
import defer * as math from "./math2.js"; // Loads everything, executes nothing

math.add; // Executes ./math2.js ./math/add.js
math.sub; // Executes ./math/sub.js
```
