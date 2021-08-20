# String is USV String

Champions: Guy Bedford
Author: Guy Bedford

Status: draft

## Problem Statement

ECMAScript strings permit unpaired UTF-16 surrogates, in contrast to many UTF implementations.

On the other hand, WebIDL defines [USVString](https://heycam.github.io/webidl/#idl-USVString) as lists of Unicode Scalar Values, which includes only the allowed UTF code point ranges [0, 0xD7FF] or [0xE000, 0x10FFFF], explicitly excluding unpaired surrogate values.

In addition, the Web Assembly [Interface Types]() proposal also restricts string values to lists of Unicode Scalar Values, [as polled in a recent CG meeting](https://github.com/WebAssembly/meetings/blob/main/main/2021/CG-08-03.md).

Since the interfacing of JavaScript strings with platform and Web Assembly APIs is a highly common use case, there is a regular need for string validations both within the platform and for certain userland use case scenarios.

## Proposal

The proposal is to define in ECMA-262 a static `String` method to verify if a given ECMAScript string is a valid USV String or not.

As a highly common scenario for interfaces between WebIDL and Wasm, this should ease certain integration scenarios that can then decide to throw or run a conversion as necessary, without having to incur custom conversion code from the start.

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
