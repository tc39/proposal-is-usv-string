# String is USV String

Champions: Guy Bedford
Author: Guy Bedford

Status: draft

## Problem Statement

[ECMAScript string values](https://262.ecma-international.org/12.0/#sec-terms-and-definitions-string-value) are a finite ordered sequence of zero or more 16-bit unsigned integer values, where each integer value in the sequence usually represents a single 16-bit code unit of UTF-16 text. However, ECMAScript does not place any restrictions or requirements on the integer values except that they must be 16-bit unsigned integers, hence ill-formed 16-bit code unit sequences containing unpaired surrogate code units (`0xD800..0xDFFF` not part of a high surrogate `0xD800..0xDBFF` followed by a low surrogate `0xDC00..0xDFFF`) are permitted during construction and string processing. In WebIDL, this concept maps to the [DOMString](https://heycam.github.io/webidl/#idl-DOMString) type.

In contrast, WebIDL also defines the [USVString](https://heycam.github.io/webidl/#idl-USVString) type as the set of all possible sequences of [Unicode Scalar Values](http://www.unicode.org/glossary/#unicode_scalar_value), which are all of the Unicode code points apart from the surrogate code points (`U+0000..U+D7FF ∧ U+E000..U+10FFFF` where surrogate *pairs* map to `U+10000..U+10FFFF`), representing well-formed Unicode text as typically consumed by for example file system and networking APIs.

In addition, the WebAssembly [Interface Types](https://github.com/WebAssembly/interface-types) proposal also restricts string values to lists of Unicode Scalar Values, [as polled in a recent CG meeting](https://github.com/WebAssembly/meetings/blob/main/main/2021/CG-08-03.md).

Since interfacing JavaScript strings with platform and WebAssembly APIs is a highly common use case and because conversion from `DOMString` to `USVString` is lossy (common options are to replace unpaired surrogates or to throw an error), there is a regular need for string validation both within the platform and for certain userland use case scenarios.

## Proposal

The proposal is to define in ECMA-262 a static `String` method to verify if a given ECMAScript string is a valid USV String or not.

As a highly common scenario for interfaces between WebIDL and Wasm, this should ease certain integration scenarios that can then decide to throw or run a conversion as necessary, without having to incur custom conversion code from the start.

## Algorithm

The validation algorithm is effectively the standard UTF-16 validation algorithm, iterating through the string and pairing UTF-16 surrogates, failing validation for any unpaired surrogates
or invalid surrogate prefix codes.

The equivalent algorithm in JavaScript is likely something along the lines of:

```js
let i = 0;
while (i < str.length) {
  const surrogatePrefix = str.charCodeAt(i) & 0xFC00;
  // Non-surrogate code point, single increment
  if (surrogatePrefix < 0xD800) {
    i += 1;
  }
  // Surrogate start
  else if (surrogatePrefix === 0xD800) {
    // Validate surrogate pair, double increment
    if ((decoded.charCodeAt(i + 1) & 0xFC00) !== 0xDC00)
      return false;
    i += 2;
  }
  else {
    // Out-of-range surrogate prefix (above 0xD800)
    return false;
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

### Are the primary benefits performance?

While the proposal is not entirely for performance reasons, it should hopefully still enable a builtin method that could be faster than user validation.

In future it might even be possible for a bit state to be associated with string data types and maintained through string functions to make the check entirely zero cost, but that would be entirely up to implementations and not anything within the reach of this specification.
