---
layout: package
title: util-array-object-or-both
---

## Purpose

When we give the user ability to choose their preference out of: `array`, `object` or `any`, we want to:

- Allow users to input the preference in many ways: for example, for `array`, we also want to accept: `Arrays`, `add`, `ARR`, `a`. Similar thing goes for options `object` and `any`.
- Normalise the choice - recognise it and set it to one of the three following strings: `array`, `object` or `any`. This is necessary because we want set values to use in our programs. You can't have five values for `array` in an IF statement, for example.
- When a user sets the preference to unrecognised string, we want to `throw` a meaningful error message. Technically this will be achieved using an options object.
- Enforce lowercase and trim and input, to maximise the input possibilities

| <br>               | Assumed to be an array-type | object-type | either type  |
| ------------------ | --------------------------- | ----------- | ------------ |
| **Input string:**  | `array`                     | `object`    | `any`        |
| <br>               | `arrays`                    | `objects`   | `all`        |
| <br>               | `arr`                       | `obj`       | `everything` |
| <br>               | `aray`                      | `ob`        | `both`       |
| <br>               | `arr`                       | `o`         | `either`     |
| <br>               | `a`                         |             | `each`       |
| <br>               |                             |             | `whatever`   |
| <br>               |                             |             | `e`          |
| <br>               | `----`                      | `----`      | `----`       |
| **Output string:** | `array`                     | `object`    | `any`        |

{% include "btt.njk" %}

## API

API is simple - just pass your value through this library's function. If it's valid, it will be normalised to either `array` or `object` or `any`. If it's not valid, an error will be thrown.

| Input argument | Type         | Obligatory? | Description                                                                 |
| -------------- | ------------ | ----------- | --------------------------------------------------------------------------- |
| `input`        | String       | yes         | Let users choose from variations of "array", "object" or "both". See above. |
| `opts`         | Plain object | no          | Optional Options Object. See below for its API.                             |

Options object lets you customise the `throw`n error message. It's format is the following:

    ${opts.msg}The ${opts.optsVarName} was customised to an unrecognised value: ${str}. Please check it against the API documentation.

| `options` object's key | Type   | Obligatory? | Default                                               | Description                                    |
| ---------------------- | ------ | ----------- | ----------------------------------------------------- | ---------------------------------------------- |
| `msg`                  | String | no          | `` | Append the message in front of the thrown error. |
| `optsVarName`          | String | no          | `given variable`                                      | The name of the variable we are checking here. |

For example, set `optsVarName` to `opts.only` and set `msg` to `ast-delete-key/deleteKey(): [THROW_ID_01]` and the error message `throw`n if user misconfigures the setting will be, for example:

    ast-delete-key/deleteKey(): [THROW_ID_01] The variable "opts.only" was customised to an unrecognised value: sweetcarrots. Please check it against the API documentation.

{% include "btt.njk" %}

## Use

```js
// require this library:
const arrObjOrBoth = require('util-array-object-or-both')
// and friends:
const clone = require('lodash.clonedeep')
const checkTypes = require('check-types-mini')
const objectAssign = require('object-assign')
// let's say you have a function:
function myPrecious (input, opts) {
  // now you want to check your options object, is it still valid after users have laid their sticky paws on it:
  // define defaults:
  let defaults = {
    lalala: null,
    only: 'object' // <<< this is the value we're particularly keen to validate, is it `array`|`object`|`any`
  }
  // clone the defaults to safeguard it, and then, object-assign onto defaults.
  // basically you fill missing values with default-ones
  opts = objectAssign(clone(defaults), opts)
  // now, use "check-types-mini" to validate the types:
  checkTypes(opts, defaults,
    {
      // give a meaningful message in case it throws,
      // customise the library `check-types-mini`:
      msg: 'my-library/myPrecious(): [THROW_ID_01]',
      optsVarName: 'opts',
      schema: {
        lalala: ['null', 'string'],
        only: ['null', 'string']
      }
    }
  )
  // by this point, we can guarantee that opts.only is either `null` or `string`.
  // if it's a `string`, let's validate is its values among accepted-ones:
  opts.only = arrObjOrBoth(opts.only, {
    msg: 'my-library/myPrecious(): [THROW_ID_02]',
    optsVarName: 'opts.only'
  })
  // now we can guarantee that it's either falsey (undefined or null) OR:
  //   - `object`
  //   - `array`
  //   - `any`

  // now you can use `opts.only` in your function safely.
  ...
  // rest of the function...
}
```

{% include "btt.njk" %}

## Critique

You may ask, why on Earth you would need a package for a such thing? It's not very universal to be useful for society, is it?

Actually, it is.

We discovered that when working with AST's, you often need to tell your tools to process (traverse, delete, and so on) EITHER objects OR arrays or both. That's where this library comes in: standardise the choice (from three options) and relieve the user from the need to remember the exact value.

We think the API should accept a very wide spectrum of values, so users would not even need to check the API documentation - they'd just describe what they want, in plain English.

I'm going to use it in:

- [ast-monkey](/os/ast-monkey/)
- [json-variables](/os/json-variables/)
- [ast-delete-key](/os/ast-delete-key/)

and others. So, it's not that niche as it might seem!

{% include "btt.njk" %}
