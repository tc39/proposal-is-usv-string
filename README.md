# String is USV String

Champions: Guy Bedford
Author: Guy Bedford

Status: draft

## Problem Statement

[ECMAScript string values](https://262.ecma-international.org/12.0/#sec-terms-and-definitions-string-value) are a finite ordered sequence of zero or more 16-bit unsigned integer values, where each integer value in the sequence usually represents a single 16-bit code unit of UTF-16 text. However, ECMAScript does not place any restrictions or requirements on the integer values except that they must be 16-bit unsigned integers, hence ill-formed 16-bit code unit sequences containing unpaired surrogate code units (`0xD800..0xDFFF` not part of a high surrogate `0xD800..0xDBFF` followed by a low surrogate `0xDC00..0xDFFF`) are permitted during construction and string processing. In WebIDL, this concept maps to the [DOMString](https://heycam.github.io/webidl/#idl-DOMString) type.

In contrast, WebIDL also defines the [Unicode Scalar Value](http://www.unicode.org/glossary/#unicode_scalar_value) (USV) string type [USVString](https://heycam.github.io/webidl/#idl-USVString) as the set of all possible sequences of Unicode Scalar Values, which are all of the Unicode code points apart from the surrogate code points (`U+0000..U+D7FF ∧ U+E000..U+10FFFF` where surrogate *pairs* map to `U+10000..U+10FFFF`), representing well-formed Unicode text as typically consumed by for example file system and networking APIs.

The WebAssembly [Interface Types](https://github.com/WebAssembly/interface-types) proposal also restricts string values to lists of Unicode Scalar Values, [as polled in a recent CG meeting](https://github.com/WebAssembly/meetings/blob/main/main/2021/CG-08-03.md).

Interfacing JavaScript strings with platform and WebAssembly APIs is a common use case that therefore suffers from conversion issues. In particular because conversion from `DOMString` to `USVString` is lossy (common options are to replace unpaired surrogates or to throw an error) there is a regular need for string validation both within the platform and for certain userland use case scenarios.

## Proposal

The proposal is to define in ECMA-262 a static `String` method to verify if a given ECMAScript string is a valid USV String or not.

As a highly common scenario for interfaces between WebIDL and Wasm, this should ease certain integration scenarios that can then decide to throw or run a conversion as necessary, without having to incur custom conversion code from the start.

## Algorithm

The validation algorithm is effectively the standard UTF-16 validation algorithm, iterating through the string and pairing UTF-16 surrogates, failing validation for any unpaired surrogates.

In JavaScript this can be achieved with the regular expression test:

```js
!/\p{Surrogate}/u.test(str);
```

Or more explicitly an algorithm along the lines of:

```js
let i = 0;
while (i < str.length) {
  const isSurrogate = (str.charCodeAt(i) & 0xF800) == 0xD800;
  if (!isSurrogate) {
    i += 1;
  } else {
    const isHighSurrogate = str.charCodeAt(i) < 0xDC00;
    if (isHighSurrogate) {
      const isFollowedByLowSurrogate = i + 1 < str.length && (str.charCodeAt(i + 1) & 0xFC00) == 0xDC00;
      if (isFollowedByLowSurrogate) {
        i += 2;
      } else {
        return false; // unpaired high surrogate
      }
    } else {
      return false; // unpaired low surrogate
    }
  }
}
return true;
```

## FAQs

### Isn't this possible today without needing a builtin API?

The two major use cases are for integration with other specifications and for userland code that for example interfaces with Web Assembly.

For users, it avoids having to write custom validators when dealing with string input in various forms, providing instead a full correct platform API that can allow easily determining the USV guarantee / invariant to apply for subsequent processing.

For integration with other platform specifications, having the ability to reference an ECMA-262 specification method for validation could also make integration easier where many APIs are now unifying on USV strings as a standard. For example, [such a method came up as a need for integration with the TextEncoder API previously](https://github.com/whatwg/encoding/issues/174) that a specification like this would be able to assist with.

### Why is the proposal to add a static String method?

Making it a static method on `String` seems like the safest home for such a method. Adding custom methods to `String.prototype` is likely quite risky, but could be considered as well.

### What about performance?

Performance optimizations are up to implementations and are not the primary motivation for this specification, yet enabling a builtin method that is faster than validation in userland is desirable. Perhaps well-formedness state could be cached per string and propagated through string operations to avoid the need for a linear scan where possible, say where strings are already lists of Unicode Scalar Values, while propagating state in operations involving strings containing inner unpaired surrogates will likely still require (re)scans.
