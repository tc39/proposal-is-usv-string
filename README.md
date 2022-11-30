# Well-Formed Unicode Strings

Champions: Guy Bedford, Bradley Farias, Michael Ficarra

Status: stage 3

## Problem Statement

[ECMAScript string values](https://tc39.es/ecma262/multipage/overview.html#sec-terms-and-definitions-string-value) are a finite ordered sequence of zero or more 16-bit unsigned integer values (in Unicode terms: individual UTF-16 code units). Unlike Unicode text, which is a sequence of Unicode Scalar Values that excludes the surrogate code points, ECMAScript does not place any constraints on the code units except that they must be 16-bit unsigned integers. Hence, not all sequences of UTF-16 code units in ECMAScript strings represent Unicode text. More precisely, the invariant that code units in the range `0xD800..0xDBFF` (leading surrogates) and `0xDC00..0xDFFF` (trailing surrogates) must appear paired and in order is not eagerly enforced for compatibility reasons. Strings with unpaired or out-of-order surrogates are called *ill-formed*.

In WebIDL, potentially ill-formed Strings are referred to using the [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString) type. Well-formed strings are referred to using the [USVString](https://webidl.spec.whatwg.org/#idl-USVString) type. Due to their asymmetric value spaces, conversions from `DOMString` to `USVString` are lossy, whereas `DOMString` can represent any `USVString`. Upon conversion from `DOMString` to `USVString`, common options are to replace unpaired surrogates with the replacement character (`U+FFFD`) or to throw an error. Where the side-effect of silent mutation or erroring is undesired, mixed UTF-8/16 systems preserve integrity instead by utilizing [WTF-8](https://simonsapin.github.io/wtf-8/). JSON preserves `DOMString` through the use of escape sequences.

The WebAssembly [Component Model](https://github.com/WebAssembly/component-model) mandates sequences of Unicode Scalar Values (`USVString`) on component boundaries, choosing to eagerly sanitize `DOMString`s including when integrating with JavaScript ([see FAQ](#how-does-this-proposal-relate-to-the-webassembly-component-model)). Independently, there is a significant number of data encodings, network interfaces, filesystem interfaces, compile-to-JS programming languages, etc. designed for Unicode text (`USVString`) with no requirement to support `DOMString`. Therefore, interfacing `DOMString`s with such APIs is a common use case that suffers from the outlined conversion burdens. To detect respectively address the side-effect of silent data mutation or thrown errors early, there is thus a need for manual string validation and sanitization both within the platform and for certain userland use case scenarios.

## Proposal

The proposal is to define in ECMA-262 a method to verify if a given ECMAScript string is well-formed or not. As a highly common scenario for interfaces between JavaScript/web APIs and those that operate on Unicode text, this test should be as efficient as possible, ideally scaling independently from the length of the string. In addition to improved performance, this method will also increase the clarity for readers of code where this test is being performed &mdash; especially those without extensive Unicode or regular expression knowledge.

The proposal also adds a method to ensure that a String is well-formed by replacing any lone or out-of-order surrogates, if any are present, with U+FFFD (REPLACEMENT CHARACTER). This operation mimics the operation already performed within various web and non-ECMAScript APIs, and reduces the chance that a consumer who would otherwise have to write this method does so in an incompatible or incorrect way.

## Algorithm

The validation algorithm is effectively the standard UTF-16 validation algorithm, iterating through the string and pairing UTF-16 surrogates, failing validation for any unpaired surrogates.

In JavaScript this can be achieved with the regular expression test:

```js
!/\p{Surrogate}/u.test(str);
```

Or more explicitly using an algorithm along the lines of:

```js
function isWellFormed(str) {
  for (let i = 0; i < str.length; ++i) {
    const isSurrogate = (str.charCodeAt(i) & 0xF800) == 0xD800;
    if (!isSurrogate) {
      continue;
    }
    const isLeadingSurrogate = str.charCodeAt(i) < 0xDC00;
    if (!isLeadingSurrogate) {
      return false; // unpaired trailing surrogate
    }
    const isFollowedByTrailingSurrogate = i + 1 < str.length && (str.charCodeAt(i + 1) & 0xFC00) == 0xDC00;
    if (!isFollowedByTrailingSurrogate) {
      return false; // unpaired leading surrogate
    }
    ++i;
  }
  return true;
}
```

Unfortunately, these user-land implementations require iterating the entire string.

## Prior Art

* WebIDL [USVString](https://heycam.github.io/webidl/#idl-USVString)
* Lone surrogate handling [discussion on WhatWG/encoding](https://github.com/whatwg/encoding/issues/174)
* Node.js [`util.toUSVString`](https://nodejs.org/dist/latest-v16.x/docs/api/util.html#util_util_tousvstring_string)

## FAQs

### Isn't this possible today without needing a builtin API?

It is possible to answer the question, but it is not possible to do it in sub-linear time.

Performance optimizations are up to implementations and are not guaranteed by this specification, yet enabling a builtin method that is faster than validation in userland is desirable. Perhaps well-formedness state could be cached per string and propagated through string operations to avoid the need for a linear scan where possible, say where strings are already lists of Unicode Scalar Values, while propagating state in operations involving strings containing inner unpaired surrogates will likely still require (re)scans.

### Are consumers going to do anything other than convert when they encounter ill-formed strings? If so, why not provide only a conversion method with a fast path for well-formed strings?

Consumers may want to throw/error when encountering ill-formed strings. Also, consumers may want to defer the conversion or the error until later when the String is actually interpreted as Unicode text. These use cases justify the test-only method.

### How does this proposal relate to the WebAssembly Component Model?

The WebAssembly [Component Model](https://github.com/WebAssembly/component-model) motivates this proposal in part, both in terms of functionality and timing. Unlike ECMAScript and similar languages like Java, which use `DOMString` respectively identical semantics to represent language-level `String`s, WebIDL, which [recommends to use `DOMString` when in doubt](https://webidl.spec.whatwg.org/#idl-USVString), and JSON, which preserves `DOMString` [through escape sequences](https://github.com/tc39/proposal-well-formed-stringify), the authors of the Component Model [obtained majority agreement](https://github.com/WebAssembly/meetings/blob/main/main/2021/CG-08-03.md#discussions) to allow only `USVString` on component boundaries, including when both producer and receiver prefer `DOMString`. The decision extends to the Component Model's planned [Web platform / ESM integration](https://github.com/WebAssembly/component-model/blob/main/design/high-level/Goals.md), increasing the usefulness of this proposal given that both new as well as existing code originally unconcerned with `USVString` is affected. In particular once JavaScript modules are upgraded to or otherwise replaced with WebAssembly components via `import`s, manual checks will become necessary to detect and work around now materializing silent mutation where undesired. As a result, such checks will be needed more frequently than pre Component Model, in turn emphasizing the importance of this proposal's suggested performance optimizations. The Component Model is currently in [phase 1](https://github.com/WebAssembly/proposals) of the WebAssembly process and there is ongoing debate whether its choices are desirable, including a [detailed objection](https://www.assemblyscript.org/standards-objections.html#component-model-2022-09) with more background. A related proposal for [Reference-Typed Strings for WebAssembly](https://github.com/WebAssembly/stringref) exists, that, contrary to the Component Model, preserves integrity by utilizing [WTF-8](https://simonsapin.github.io/wtf-8/). It cannot be ruled out that the Component Model's `string` and Reference-Typed Strings' `stringref` align in the future for consistency reasons, then either subtracting from or adding to this proposal's problem statement.
