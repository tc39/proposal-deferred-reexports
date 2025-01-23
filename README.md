# Deferred re-exports

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

As part of the proposal, we considered ([2023-11 slides](https://docs.google.com/presentation/d/1l-H2ntEDZGAWvtuOup1TJdylZsV1epKVSejVM-GwHLU), [2024-04 `export ... from` slides](https://docs.google.com/presentation/d/1iM5cRgdRXLWLq_GxgRvzYmUTXEK6gzH_8QNgLKMmv7o)) adding similar functionality to `export ... from` declaration. However, `export ... from` declaration have the opportunity to also skip _loading_ of unused modules, and we thus when advancing `import defer` to Stage 2.7 we decided ([2024-04 `import defer` slides](https://docs.google.com/presentation/d/1oPEF8nA9Iq5cAqjN-FqMigNNfz6lWCUbNfIsEjRXf4Y)) to keep the equivalend `export ... from` feature behind as a separate proposal.

## Problem statement

A common pattern that libraries adopt to improve DX of their consumers is to have a single entry point that re-exports all the public APIs. This however has a problem: it leads to a lot of unused code being included in the module graph.

Some tools workaround this problem using a technique called "tree-shaking": they trace the dependencies of the indivudual bindings exported by a module, and remove the transivite imports (or re-exports) that are only needed for unused exports. However, this operation is not always possible due to side effects that may caused by modules being loaded: different tools choose different trade-offs between code size and correctness, often leading to under-optimization of web applications or to difficult-to-debug issues caused by non-pure modules not being executed as expected. This technique is also _not_ possible when running ESM directly in the browser, as it requires whole-program analysys.

This proposal aims to address the problem by allowing libraries to mark re-exports as "ignore if unused", defining:
1. clear semantics to follow, rather than relying on tooling-defined heuristics;
2. semantics that can also be implemented by native JS platforms to avoid loading unused code;
3. an integration with the `import defer` proposal, to allow these re-exports to benefit from the same "after loading, only execute if actually needed" semantics.
