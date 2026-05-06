# Deferred re-exports

## Status

Champion(s): _Nicolò Ribaudo_, _Caio Lima_

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

This proposal aims to allow libraries to continue using this popular pattern while removing the drawbacks that it comes with.

It does so by allowing libraries to mark re-exports as "ignore if unused", defining:
1. clear semantics to follow, rather than relying on tooling-defined heuristics;
2. semantics that can also be implemented by native JS platforms to avoid loading unused code;
3. an integration with the `import defer` proposal, to allow these re-exports to benefit from the same "after loading, only execute if actually needed" semantics.

This proposal uses the syntax below, parallel to the [import defer](https://github.com/tc39/proposal-defer-import-eval/) proposal:
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

`export defer * from './foo.js` is not supported and not planned to be supported, because we require that deferred reexport names can be explicitly known without further dependency loading, as a requirement of lazy network loading.

### Namespace imports

On module namespace objects, `export defer` will "deoptimize" and eagerly load all modules, but still allow lazy execution of those modules:

```js
// math.js
export defer { add } from "./math/add.js";
export defer { sub } from "./math/sub.js";
export { mul } from "./math/mul.js";
console.log('executed math');
```
```js
// index.js
import * as math from "./math.js"; // Loads everything (both ./math.js and ./math/mul.js), logs 'executed math'

math.add; // Executes ./math/add.js
math.sub; // Executes ./math/sub.js
```

A similar de-optimization happens with `export * from`, which will cause all the optional dependencies used for exports other than `default` to be eagerly loaded:

```js
// math.js
export defer { add as default } from "./math/add.js";
export defer { sub } from "./math/sub.js";
export { mul } from "./math/mul.js";
```
```js
// index.js
export * from "./math.js"; // Loads (./math.js, ./math/mul.js and ./math/add.js), does not load ./math/add.js
```


### Integration with `import defer`

The `import defer` proposal established that the `defer` keyword means "only execute this module when I actually need it", when using _namespace_ imports. When interacting with `export defer`, this allows for lazily executing the wrapper modules as well:

```js
// math.js
export defer { add } from "./math/add.js";
export defer { sub } from "./math/sub.js";
export { mul } from "./math/mul.js";
console.log('executed math');
```
```js
// index.js
import * as math from "./math.js"; // Loads everything (both ./math.js and ./math/mul.js), no execution at all

math.add; // Executes ./math/add.js, logs 'executed math'
math.sub; // Executes ./math/sub.js
```

All dependencies are still loaded upfront, and deferred modules that use top-level await are pre-executed (like when using the `import defer` proposal alone), so that they are synchronously available upon request.

### Comparison with `package.json#exports`

Originally introduced by Node.js but now recognized by multiple build tools, `package.json#exports` allows defining the entry points of a JavaScript package ([docs](https://nodejs.org/api/packages.html#package-entry-points)).

A library, in this example called `react`, could have a main entrypoint that is environment-agnostic and then have multiple entry points for specialized features:
```json
{
  "exports": {
    ".": "./lib/index.js",
    "./dom": "./lib/dom.js",
    "./ssr": "./lib/ssr.js"
  }
}
```

When importing it, users can then use the "good-looking" entry point names rather than having to import the library's internal files:
```js
import { createElement } from "react"; // Imports react/lib/index.js
import { render } from "react/dom"; // Imports react/lib/dom.js
```

This allow libraries to provide separate entrypoints for separate features, without requiring users to use deep paths into the library's chosen file structure. It gives to the library's users a similar experience as if all the files that users are meant to import were all living in the root folder of such library. `package.json#exports` work enteirely at resolution time.

`export defer` partially overlaps with `package.json#exports`: both of them allow importers to import _parts_ of a library without having to write multiple complex paths. However, they have different trade-offs and are not interchangeable, but can work together:
- `package.json#exports` only works at the "Node.js package" level. Complex applications often have internal folders that are conceptually internal libaries, with their own entry point. `export defer`, being a language feature, is not tied to the "package" concept from Node.js, and can be used wherever within the application.
- `export defer` would work natively in browsers, while `package.json#exports` is a feature specific to how Node.js resolution works.
- `package.json#exports` allows defining which JavaScript files within a library are internal and which ones are part of the public API, preventing the library's users from importing internal files. `export defer` does not related to this use case.

There are existing popular libraries that already mix `package.json#exports` and barrel files. One example is [`msw`](https://github.com/mswjs/msw) ("Mock Service Worker"), which:
- defines environment-specific entry points using `package.json#exports` (e.g. `msw/node` and `msw/browser`) — [msw/package.json](https://github.com/mswjs/msw/blob/de188887793fcc1956f4e506459fe3db0a13dabf/package.json)
- in each entry point, it re-exports multiple tools using `export { ... } from` — [msw/src/core/index.ts](https://github.com/mswjs/msw/blob/de188887793fcc1956f4e506459fe3db0a13dabf/src/core/index.ts)
