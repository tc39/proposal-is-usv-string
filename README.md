# Well-Formed Unicode Strings

Champions: Guy Bedford, Bradley Farias, Michael Ficarra

Status: stage 1

## Problem Statement

[ECMAScript string values](https://tc39.es/ecma262/multipage/overview.html#sec-terms-and-definitions-string-value) are a finite ordered sequence of zero or more 16-bit unsigned integer values. However, ECMAScript does not place any restrictions or requirements on the integer values except that they must be 16-bit unsigned integers. In *well-formed* strings, each integer value in the sequence represents a single 16-bit code unit of UTF-16-encoded Unicode text. However, not all sequences of UTF-16 code units represent UTF-16-encoded Unicode text. In well-formed strings, code units in the range `0xD800..0xDBFF` (leading surrogates) and `0xDC00..0xDFFF` (trailing surrogates) must appear paired and in order. Strings with unpaired or out-of-order surrogates are *ill-formed*.

In WebIDL, possibly-ill-formed Strings are referred to using the [DOMString](https://webidl.spec.whatwg.org/#idl-DOMString) type. But for interfaces which operate on Unicode text, WebIDL also defines the [USVString](https://webidl.spec.whatwg.org/#idl-USVString) type as the set of all possible sequences of Unicode Scalar Values, which are all of the Unicode code points apart from the surrogate code points (`U+0000..U+D7FF` and `U+E000..U+10FFFF`), representing Unicode text.

WebAssembly [Interface Types](https://github.com/WebAssembly/interface-types) require well-formed strings, as do some compile-to-JS programming languages, many data encodings, network interfaces, filesystem interfaces, etc. Interfacing JavaScript strings with such APIs is a common use case that therefore suffers from conversion burdens. In particular because conversion from `DOMString` to `USVString` is lossy (common options are to replace unpaired surrogates or to throw an error) there is a regular need for string validation both within the platform and for certain userland use case scenarios.

## Proposal

The proposal is to define in ECMA-262 a method to verify if a given ECMAScript string is well-formed or not. As a highly common scenario for interfaces between JavaScript/web APIs and those that operate on Unicode text, this test should be as efficient as possible, ideally scaling independently from the length of the string.

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

### Are consumers going to do anything other than convert when they encounter ill-formed strings? If so, why not just provide a conversion method with a fast path for well-formed strings?

Consumers may want to throw/error when encountering ill-formed strings. Also, consumers may want to defer the conversion or the error until later when the String is actually interpreted as Unicode text. These use cases justify the test-only method. Since it's easy to build a conversion method given the test-only method, the conversion method is omitted from this proposal at this time. See [#13](https://github.com/tc39/proposal-is-usv-string/issues/13) for discussion.
