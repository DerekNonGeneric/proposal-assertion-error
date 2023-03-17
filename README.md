# `AssertionError`

Rough draft proposal for an `AssertionError` exception thrown to indicate the failure of an
assertion has occurred.

## Status

This proposal has not yet been introduced to TC39.

## Authors

- [@DerekNonGeneric](https://github.com/DerekNonGeneric) (Derek Lewis, AMPHTML / OpenINF)
- Spec. Guest(s):
  - [@boneskull](https://github.com/boneskull) (Chris Hiller, Mocha)
    
 ### Champion Group
    
  - Invited Expert(s):
    - TBD
  - Delegate(s):
    - TBD

## Motivation

Today, a very common design pattern (especially in assertion library code) is to
throw an `AssertionError` for any failed assertions.

In particular, the following popular assertion libraries do this:

- [Chai](https://github.com/chaijs/chai)
- [Expect](https://jestjs.io/docs/expect)
- [Node.js core `assert` module](https://nodejs.org/dist/latest/docs/api/assert.html#assert_assert)
- [See Related](#related)

Ideally, a standard `AssertionError` from any assertion library would be used by
the error reporters of test frameworks and runtime error loggers to display the
difference between what was expected and what the assertion saw for providing
rich error reporting such as displaying a pretty printed diff.

However, due to the various assertion libraries using disparate `AssertionError`
classes, with each providing varying degrees of contextual information, a
standard API for assertion error message pretty printing does not yet exist.

This proposal introduces a 7th standard error type to the language to provide a
consistent API necessary to enable assertion library-agnostic error handlers
with the contextual information necessary for rich error reporting.

There are several existing modules with similar ideas of how the interface for
this error is expected to look:

- [npm: `assertion-error`](https://www.npmjs.com/package/assertion-error)
- [npm: `assertion-error-diff`](https://www.npmjs.com/package/assertion-error-diff)
- [See Related](#related)

## Proposal

Today, it is very common for unit testing frameworks to have assertion methods
like `assertNull` write code like:

```mjs
// file: check.mjs
export function check(actual) {
  return {
    is: function (expect, message) {
      if (actual !== expect)
        throw new AssertionError({
          actual: actual,
          expected: expect,
          message: message,
        });
    },
  };
}

export default check;
```

This would be used as follows:

```mjs
import check from './check.mjs';

export function assertNull(value) {
  check(value).is(null, `${value} is not null`);
}

export default assertNull;
```

<!--
Reference implementation for runner (WIP):

```mjs
try {
  test();
  results.pass();
} catch (e if e instanceof Skip) {
  console.log("SKIPPED");
  results.skip();
} catch (e if e instanceof AssertionError) {
  console.log("FAILED" + (e.message ? (": " + e.message) : ""));
  results.fail(e.message);
} catch (e) {
  console.log("ERROR: " + e);
  results.error(e);
}
```
-->

The `AssertionError` class is not only for pretty printing error messages. It
plays a key role in applying &ldquo;Design by Contract&rdquo;, which often means
performing runtime assertions.

> An assertion specifies that a program satisfies certain conditions at
> particular points in its execution. There are three types of assertion:
>
> - Preconditions: Specify conditions at the start of a function.
>
> - Postconditions: Specify conditions at the end of a function.
>
> - Invariants: Specify conditions over a defined region of a program.
>
> &mdash;https://ptolemy.berkeley.edu/~johnr/tutorials/assertions.html

The examples below demonstrate &ldquo;Postconditions&rdquo;.

```mjs
assert.species = (pokemon, species, message) => {
  const actual = pokemon.template.species;
  if (actual === species) return;
  throw new AssertionError({
    actual,
    expected: species,
    message:
      message || `Expected ${pokemon} species to be ${species}, not ${actual}.`,
    stackStartFunction: assert.species,
  });
};
```

However, none of the properties of the options object would be mandatory.

```mjs
assert.fullHP = function (pokemon, message) {
  if (pokemon.hp === pokemon.maxhp) return;
  throw new AssertionError({
    message:
      message ||
      `Expected ${pokemon} to be fully healed, not at ${pokemon.hp}/${pokemon.maxhp}.`,
    stackStartFunction: assert.fullHP,
  });
};
```

The more contextual information, the richer the error reports. So, if we had an
operator property&hellip;

```mjs
assert.heals = (pokemon, fn, message) => {
  const prevHP = pokemon.hp;
  fn();
  if (pokemon.hp > prevHP) return;
  throw new AssertionError({
    actual: pokemon.hp,
    expected: `${prevHP}`,
    operator: '>',
    message: message || `Expected ${pokemon} to be healed.`,
    stackStartFunction: assert.heals,
  });
};
```

&hellip; it would enable us to construct highly descriptive reports that include
comparison result data.

[Attribution](https://github.com/Fercardo/Inferno/blob/HEAD/test/assert.js#L61)

## Inspiration

Some notable prior art helping drive this proposal.

### Node.js Built-in Class: `assert.AssertionError`

- Extends: {errors.Error}

Indicates the failure of an assertion. All errors thrown by the `assert` module
will be instances of the `AssertionError` class.

### `new assert.AssertionError(options)`

A subclass of `Error` that indicates the failure of an assertion.

- `options` {Object}
  - `message` {string} If provided, the error message is set to this value.
  - `actual` {any} The `actual` property on the error instance.
  - `expected` {any} The `expected` property on the error instance.
  - `operator` {string} The `operator` property on the error instance.
  - `stackStartFn` {Function} If provided, the generated stack trace omits
    frames before this function.

## Ecosystem uses

### Custom assert modules

> To make it possible to combine assertions from different modules in one test
> suite, all assert methods should throw an `AssertionError` that has properties
> for `actual` and `expected` an common API for error message pretty printing.
>
> &mdash;http://wiki.commonjs.org/wiki/Unit_Testing/1.0#Custom_Assert_Modules

### Test & validation frameworks

> Mocha supports the `err.expected` and `err.actual` properties of any thrown
> AssertionErrors from an assertion library. Mocha will attempt to display the
> difference between what was expected, and what the assertion actually saw.
>
> &mdash;https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs

### Runtime feature detection errors

> An expected error is identified by the `.expected` property on the `Error`
> object being `true`. You can use the `Log.prototype.expectedError` method to
> create an error that is marked as expected.
>
> &mdash;https://github.com/ampproject/amphtml/blob/main/docs/spec/amp-errors.md#expected-errors

## Options

All instances of `AssertionError` would contain the built-in `Error` properties
(`message` and `name`) and perhaps any of the new properties in common use
below.

### Support for new properties

|                        | `actual` | `expected` | [`operator`][] | [`messagePattern`][] | `generateMessage()` | `generatedMessage` | `diffable` | `showDiff` | `toJson()` | [`stack`][] | `stackStartFn()` | `stackStartFunction()` |                                  `code`                                   | `details` | `truncate` | `previous` | `negate` | `_message` | `assertion` | `fixedSource` | `improperUsage` | `actualStack` | `raw` | `statements` | `savedError` |
| :--------------------: | :------: | :--------: | :------------: | :------------------: | :-----------------: | :----------------: | :--------: | :--------: | :--------: | :---------: | :--------------: | :--------------------: | :-----------------------------------------------------------------------: | :-------: | :--------: | :--------: | :------: | :--------: | :---------: | :-----------: | :-------------: | :-----------: | :---: | :----------: | :----------: |
|        [AVA][]         |          |            |       ×        |                      |                     |                    |            |            |            |             |                  |                        |                                                                           |           |            |            |          |            |      ×      |       ×       |        ×        |       ×       |   ×   |      ×       |      ×       |
|        [Chai][]        |    ×     |     ×      |       ×        |                      |                     |                    |            |     ×      |     ×      |             |                  |                        |                                                                           |           |            |            |          |            |             |               |                 |               |       |              |              |
|  [Closure Library][]   |          |            |                |          ×           |                     |                    |            |            |            |             |                  |                        |                                                                           |           |            |            |          |            |             |               |                 |               |       |              |              |
|        [Deno][]        |    ×     |     ×      |       ×        |                      |                     |         ×          |            |            |            |      ×      |        ×         |           ×            |                                     ×                                     |     ×     |            |            |          |            |             |               |                 |               |       |              |              |
|        [Jest][]        |    ×     |     ×      |       ×        |                      |                     |                    |            |            |            |             |                  |                        |                                                                           |           |            |            |          |            |             |               |                 |               |       |              |              |
|       [Mocha][]        |    ×     |     ×      |                |                      |                     |                    |            |     ×      |            |             |                  |                        | [x](https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#error-codes) |           |            |            |          |            |             |               |                 |               |       |              |              |
| [Mozilla Assert.jsm][] |    ×     |     ×      |       ×        |                      |                     |                    |            |            |            |             |                  |                        |                                                                           |           |     ×      |            |          |            |             |               |                 |               |       |              |              |
|      [Must.js][]       |    ×     |     ×      |       ×        |                      |                     |                    |     ×      |     ×      |            |      ×      |                  |                        |                                                                           |           |            |            |          |            |             |               |                 |               |       |              |              |
|   [Node.js Assert][]   |    ×     |     ×      |       ×        |                      |                     |         ×          |            |            |            |      ×      |        ×         |           ×            |                                     ×                                     |     ×     |            |            |          |            |             |               |                 |               |       |              |              |
|     [Should.js][]      |    ×     |     ×      |       ×        |                      |          ×          |         ×          |            |            |            |      ×      |        ×         |           ×            |                                                                           |     ×     |            |     ×      |    ×     |     ×      |             |               |                 |               |       |              |              |
|        [WPT][]         |          |            |                |                      |                     |                    |            |            |            |      ×      |                  |                        |                                                                           |           |            |            |          |            |             |               |                 |               |       |              |              |

Note: [`Error.prototype.stack`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/Stack) is not supported in all browsers/versions.

[ava]: https://github.com/avajs/ava/blob/97525a97c0f1e1fc609c980f6a4e66758c26480a/lib/assert.js#L38
[chai]: https://github.com/chaijs/assertion-error/blob/HEAD/index.js#L44
[closure library]: https://github.com/google/closure-library/blob/HEAD/closure/goog/asserts/asserts.js#L54
[deno]: https://deno.land/std@0.92.0/node/assertion_error.ts#L376
[jest]: https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs
[mocha]: https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs
[must.js]: https://github.com/moll/js-must/blob/HEAD/lib/assertion_error.js#L5
[node.js assert]: https://github.com/nodejs/node/blob/HEAD/lib/internal/assert/assertion_error.js#L327
[mozilla assert.jsm]: https://searchfox.org/mozilla-central/rev/0b90e582d2f592a30713bafc55bfeb0e39e1a1fa/testing/modules/Assert.jsm#105
[should.js]: https://github.com/shouldjs/should.js/blob/master/lib/assertion-error.js
[stack]: https://github.com/tc39/proposal-error-stacks
[wpt]: https://github.com/web-platform-tests/wpt/blob/7b0ebaccc62b566a1965396e5be7bb2bc06f841f/resources/testharness.js#L3770
[`messagepattern`]: https://developer.mozilla.org/en-US/docs/Web/API/Console#using_string_substitutions
[`operator`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators
[`stack`]: https://github.com/tc39/proposal-error-stacks

### Descriptions of custom properties

|       Property       |   Type   |                                                        Meaning                                                        |
| :------------------: | :------: | :-------------------------------------------------------------------------------------------------------------------: |
|       `actual`       | unknown  |                       Set to the `actual` argument for methods such as `assert.strictEqual()`.                        |
|      `callsite`      | Function |                                        Location where the assertion happened.                                         |
|        `code`        |  string  |                     Value is always `ERR_ASSERTION` to show that the error is an assertion error.                     |
|      `details`       |  Object  |                                    The context data necessary in a single object.                                     |
|      `diffable`      | boolean  |        Whether it makes sense to compare objects granularly or even show a diff view of the objects involved.         |
|      `expected`      | unknown  |                        Set to the `expected` value for methods such as `assert.strictEqual()`.                        |
|  `generatedMessage`  | boolean  |                             Indicates if the message was auto-generated (`true`) or not.                              |
|      `message`       | string?  |                                        Message describing the assertion error.                                        |
|   `messagePattern`   | unknown  | The message pattern used to format the error message. Error handlers can use this to uniquely identify the assertion. |
|      `operator`      |  string  |                                         Set to the passed in operator value.                                          |
|      `showDiff`      | boolean  |           Same as `diffable`. Used by mocha; whether to do string or object diffs based on actual/expected.           |
|       `stack`        | unknown  |                            The stack trace at the point where this error was first thrown.                            |
|    `stackStartFn`    | Function |                       If provided, the generated stack trace omits frames before this function.                       |
| `stackStartFunction` | Function |                                Legacy name for `stackStartFn` in Node.js also in Deno.                                |
|      `toJSON()`      | Function |                               Allow errors to be converted to JSON for static transfer.                               |
|      `truncate`      | boolean  |                       Whether or not `actual` and `expected` should be truncated when printing.                       |

## Implementations in other languages

- `assert.AssertionError` has been one of Node's core modules since 2009
  https://nodejs.org/dist/latest-v15.x/docs/api/assert.html#assert_class_assert_assertionerror

- `AssertionError` has been one of Python's Standard Exception Classes since
  Python 1.5. https://www.python.org/doc/essays/stdexceptions/

- `AssertionError` in Java also accepts a `cause`
  https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/AssertionError.html

- `AssertionError` in PHP simply extends `Error` https://www.php.net/manual/en/class.assertionerror.php

- `AssertionError` in Julia (since v0.5.0)
  - optionally accepts a message https://docs.julialang.org/en/v1/base/base/#Core.AssertionError
  - is thrown from [`@assert`](https://github.com/JuliaLang/julia/blob/54c7002892d1b0260890848397e4ba1e2d2506d1/doc/src/manual/metaprogramming.md#building-an-advanced-macro) macros

## Related

Good amount of implementations in JS existence today.

### Predicates

- [fn: `isAssertionError` in Jest](https://github.com/facebook/jest/blob/HEAD/packages/jest-circus/src/formatNodeAssertErrors.ts#L185)
- [fn: `isAssertionError` in `assertion-error-diff`](https://github.com/TylorS167/assertion-error-diff/blob/HEAD/src/isAssertionError.ts#L3)

### Modules

- [npm: `assertion-error`](https://www.npmjs.com/package/assertion-error)
- [npm: `assertion-error-diff`](https://www.npmjs.com/package/assertion-error-diff)
- [node: `assert.AssertionError`](https://github.com/nodejs/node/blob/HEAD/doc/api/assert.md#class-assertassertionerror)
- [npm: `better-assert`](https://github.com/tj/better-assert)
- [npm: `callsite`](https://github.com/tj/callsite)

### Formatters

- [fn: `console.assert()` in the `console` Web API](https://developer.mozilla.org/en-US/docs/Web/API/console/assert)
- [fn: `formatNodeAssertErrors()` in `jest-circus`](https://github.com/facebook/jest/blob/HEAD/packages/jest-circus/src/formatNodeAssertErrors.ts#L41)

## Q&A

### Should this exception be thrown when an import assertion fails?

The simple answer is that it would be the _most_ appropriate error type to throw.

[Further reading&hellip;](https://github.com/tc39/proposal-import-assertions/issues/3#issuecomment-656752033)

### Why not custom subclasses of Error

While there are lots of ways to achieve the behavior of the proposal, if the
various AssertionError properties are explicitly defined by the language, debug
tooling, error reporters, and other AssertionError consumers can reliably use
this info rather than contracting with developers to construct an error properly.

## Notes

- [Implement ES stage-3 proposal: `error-cause`](https://github.com/nodejs/node/issues/38725)
