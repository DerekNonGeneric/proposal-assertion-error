# `AssertionError`

Proposal for an `AssertionError` exception thrown to indicate the failure of an
assertion has occurred.

## Status

This proposal has not yet been introduced to TC39.

Authors:

- [@DerekNonGeneric](https://github.com/DerekNonGeneric) (Derek Lewis, Indiana University)
- Champion: TBD

## Motivation

Today, a very common design pattern (especially in assertion library code)
is to throw an `AssertionError` for any failed assertions.

In particular, the following popular assertion libraries do this:

- chai
- qunit
- nodejs core assert module
- [See Related](#related)

Ideally, a standard AssertionError from any assertion library would be used by the error reporters of test frameworks and runtime error loggers to display the difference between what was expected and what the assertion saw for providing rich error reporting such a pretty printed diff.

However, due to the various assertion libraries using disparate AssertionError classes, with each providing varying degrees of contextual information, a standard API for assertion error message pretty printing does not yet exist.

This proposal introduces a 7th standard error type to the language to provide a consistent API necessary to enable assertion library-agnostic error handlers with the contextual information necessary for rich error reporting.

There are several existing modules with similar ideas of how the interface for this error is expected to look:

- [npm: `assertion-error`](https://www.npmjs.com/package/assertion-error)
- [npm: `assertion-error-diff`](https://www.npmjs.com/package/assertion-error-diff)
- [See Related](#related)

## Proposal

Today, it is very common unit testing frameworks to have assertion methods to
like `assertNull` write code like:

```ts
// file: check.mjs
function check(actual) {
  return {
    is: function (expect, message) {
      if (actual !== expect)
        throw new AssertionError({
          actual: actual,
          expected: expected,
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

Or if we had an `operator` property:

```ts
assert.species = (pokemon, species, message) => {
  const actual = pokemon.template.species;
  if (actual === species) return;
  throw new AssertionError({
    actual,
    expected: species,
    operator: '===',
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

This is not only for pretty printing error messages! We can use this data for
making the assertion itself.

```ts
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

[Attribution](https://github.com/Fercardo/Inferno/blob/HEAD/test/assert.js#L61)

<br />

---

<br />

Can be used by various exception handlers for display purposes (more descriptive
error messages):

- runtime exception handling
  - [Node.js 📦 `assert.AssertError`](https://github.com/nodejs/node/blob/HEAD/doc/api/assert.md#class-assertassertionerror)
- pretty printing errors in test frameworks
  - [`jest-circus` 📦 `formatNodeAssertErrors()`](https://github.com/facebook/jest/blob/HEAD/packages/jest-circus/src/formatNodeAssertErrors.ts#L23-L28)

## Inspiration

Some notable prior art heliping drive this proposal.

## Node.js Builtin Class: `assert.AssertionError`

- Extends: {errors.Error}

Indicates the failure of an assertion. All errors thrown by the `assert` module
will be instances of the `AssertionError` class.

## `new assert.AssertionError(options)`

A subclass of `Error` that indicates the failure of an assertion.

- `options` {Object}
  - `message` {string} If provided, the error message is set to this value.
  - `actual` {any} The `actual` property on the error instance.
  - `expected` {any} The `expected` property on the error instance.
  - `operator` {string} The `operator` property on the error instance.
  - `stackStartFn` {Function} If provided, the generated stack trace omits
    frames before this function.

<br />

---

<br />

## Ecosystem uses

### Custom assert modules

> To make it possible to combine assertions from different modules in one test
> suite, all assert methods should throw an `AssertionError` that has properties
> for `actual` and `expected` an common API for error message pretty printing.
>
> &mdash; http://wiki.commonjs.org/wiki/Unit_Testing/1.0#Custom_Assert_Modules

### Test & validation frameworks

> Mocha supports the `err.expected` and `err.actual` properties of any thrown AssertionErrors from an assertion library.
> Mocha will attempt to display the difference between what was expected, and what the assertion actually saw.
>
> &mdash; https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs

### Runtime feature detection errors

> An expected error is identified by the `.expected` property on the `Error`
> object being `true`. You can use the `Log.prototype.expectedError` method to
> create an error that is marked as expected.
>
> &mdash; https://github.com/ampproject/amphtml/blob/main/spec/amp-errors.md#expected-errors

<br />

---

<br />

## Implementations

### Options

All instances of `AssertionError` would contain the built-in `Error` properties
(`message` and `name`) and perhaps any of the new properties in common use below.

### Support for new properties

|             | `actual` | `expected` | `operator` | `messagePattern` | `generatedMessage` | `diffable` | `showDiff` | `toJson()` | `stack` | `stackStartFn()` | `stackStartFunction()` | `code`                                                                    | `details` |
| ----------- | -------- | ---------- | ---------- | ---------------- | ------------------ | ---------- | ---------- | ---------- | ------- | ---------------- | ---------------------- | ------------------------------------------------------------------------- | --------- |
| [Chai.js][] | X        | X          | X          |                  |                    |            | X          | X          |         |                  |
| [Closure][] |          |            |            | X                |
| [Deno][]    | X        | X          | X          |                  | X                  |            |            |            | X       | X                | X                      | X                                                                         | X         |
| [Jest][]    | X        | X          | X          |
| [Mocha][]   | X        | X          |            |                  |                    |            | X          |            |         |                  |                        | [X](https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#error-codes) |           |
| [Must.js][] | X        | X          | X          |                  |                    | X          | X          |            | X       |
| [Node.js][] | X        | X          | X          |                  | X                  |            |            |            | X       | X                | X                      | X                                                                         | X         |

[chai.js]: https://github.com/chaijs/assertion-error/blob/HEAD/index.js#L44
[closure]: https://github.com/google/closure-library/blob/HEAD/closure/goog/asserts/asserts.js#L54
[deno]: https://deno.land/std@0.92.0/node/assertion_error.ts#L376
[jest]: https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs
[mocha]: https://github.com/mochajs/mocha/blob/HEAD/docs/index.md#diffs
[must.js]: https://github.com/moll/js-must/blob/HEAD/lib/assertion_error.js#L5
[node.js]: https://github.com/nodejs/node/blob/HEAD/lib/internal/assert/assertion_error.js#L327

### Descriptions of custom properties

| Property             | Type     | Meaning                                                                                                               |
| -------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| `actual`             | unknown  | Set to the `actual` argument for methods such as `assert.strictEqual()`.                                              |
| `callsite`           | function | Location where the assertion happened.                                                                                |
| `code`               | string   | Value is always `ERR_ASSERTION` to show that the error is an assertion error.                                         |
| `details`            | Object   | The context data necessary in a single object.                                                                        |
| `diffable`           | boolean  | Whether it makes sense to compare objects granularly or even show a diff view of the objects involved.                |
| `expected`           | unknown  | Set to the `expected` value for methods such as `assert.strictEqual()`.                                               |
| `generatedMessage`   | boolean  | Indicates if the message was auto-generated (`true`) or not.                                                          |
| `message`            | string?  | Message describing the assertion error.                                                                               |
| `messagePattern`     | unknown  | The message pattern used to format the error message. Error handlers can use this to uniquely identify the assertion. |
| `operator`           | string   | Set to the passed in operator value.                                                                                  |
| `showDiff`           | boolean  | Same as `diffable`. Used by mocha; whether to do string or object diffs based on actual/expected.                     |
| `stack`              | unknown  | The stack trace at the point where this error was first thrown.                                                       |
| `stackStartFn`       | function | If provided, the generated stack trace omits frames before this function.                                             |
| `stackStartFunction` | function | Legacy name for `stackStartFn` in Node.js also in Deno.                                                               |
| `toJSON()`           | function | Allow errors to be converted to JSON for static transfer.                                                             |

<br />

## Notes

- `assert.AssertionError` has been one of Node's core modules since 2011
  https://nodejs.org/dist/latest-v15.x/docs/api/assert.html#assert_class_assert_assertionerror

- `AssertionError` has been one of Python's Standard Exception Classes since
  Python 1.5. https://www.python.org/doc/essays/stdexceptions/
- `AssertionError` in Java also accepts a `cause` https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/AssertionError.html

<br />

---

<br />

## Related

Good amount of implementations in existence today.

### Predicates

- [fn: `isAssertionError` in Jest](https://github.com/facebook/jest/blob/HEAD/packages/jest-circus/src/formatNodeAssertErrors.ts#L185)
- [fn: `isAssertionError` in `assertion-error-diff`](https://github.com/TylorS167/assertion-error-diff/blob/HEAD/src/isAssertionError.ts#L3)

### Modules

- [npm: `assertion-error`](https://www.npmjs.com/package/assertion-error)
- [npm: `assertion-error-diff`](https://www.npmjs.com/package/assertion-error-diff)

## Q&A

### Import assertions

- Should this exception be raised when an import assertion fails?
