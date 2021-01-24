---
layout: package
title: ast-monkey
---

## Context

A single HTML tag `<td>a</td>` can be parsed into an AST (see the [playground](https://astexplorer.net/)):

```json
[
  {
    "type": "tag",
    "name": "td",
    "attribs": {},
    "children": [
      {
        "data": "a",
        "type": "text",
        "next": null,
        "startIndex": 4,
        "prev": null,
        "parent": "[Circular ~.0]",
        "endIndex": 5
      }
    ],
    "next": null,
    "startIndex": 0,
    "prev": null,
    "parent": null,
    "endIndex": 10
  }
]
```

`ast-monkey` performs operations on such nested arrays and objects like the one above.

{% include "btt.njk" %}

## The challenge

Operations on AST's — Abstract Syntax Trees — or anything deeply nested are difficult. 

**The main problem** is going "up the branch" — querying the parent and sibling nodes.

**Second problem**, AST's get VERY BIG very quickly. A single tag, `<td>a</td>`, 10 characters produced 398 characters of AST above. Enormous inputs are very hard to reason about, especially to troubleshoot, printed trees don't fit into screen.

**The first problem**, the "Going up", is often solved by putting circular references in the parsed tree, like `"parent": "[Circular ~.0]",` in the example above. The first drawback of using circular references is that it's not standard JSON, you can't even `JSON.stringify` (specialised stringification [packages](https://www.npmjs.com/package/json-stringify-safe) do exist) — everything from the algorithm up to the unit test runner are affected. The second drawback of circular references is that while they make it easier to query things, they also make it harder to amend things — you have to amend "circular extras" as well (or hope renderer will be OK, but that's only for small operations).

This program doesn't rely on circular references. It uses indexing of "breadcrumb" paths. For example, you traverse and find that node you want is index `58`, whole path being `[2, 14, 16, 58]`. You save the path down. After the traversal is done, you fetch the monkey to delete the index `58`. You can also use a `for` loop on breadcrumb index array, `[2, 14, 16, 58]` and fetch and check parent `16` and grandparent `14`. Lots of possibilities. Method [.find()](#find) searches using key or value or both, and method [.get()](#get) searches using a known index. That's the strategy.

{% include "btt.njk" %}

## Idea

Conceptually, we use two systems to mark paths in AST:

1. Our unique, number-based indexing system — each encountered node is numbered, for example, `58` (along with "breadcrumb" path, an array of integers, for example, `[2, 14, 16, 58]`). If you know the number you can get monkey to fetch you the node at that number or make amends on it.
2. [object-path](https://www.npmjs.com/package/object-path) notation, as in `foo.1.bar` (instead of `foo[1].bar`). The dot marking system is also powerful, it is used in many our programs, althouh it has some shortcomings ([no dots in key names](https://github.com/mariocasciaro/object-path/issues/96), for example).

[Traversal](#traverse) function will report both ways.

{% include "btt.njk" %}

## API

### .find()

Method `find()` can search objects by key or by value or by both and return the indexes path to an each finding.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |
| `options`      | Object   | yes         | Options object. See below.                                      |

| Options object's key | Type     | Obligatory?                              | Description                                                                                                                |
| -------------------- | -------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `key`                | String   | at least one, `key` or `val`             | If you want to search by a plain object's key, put it here.                                                                |
| `val`                | Whatever | at least one, `key` or `val`             | If you want to search by a plain object's value, put it here.                                                              |
| `only`               | String   | no (if not given, will default to `any`) | You can specify, to find only within arrays, objects or any. `any` is default and will be set if `opts.only` is not given. |

Either `opts.key` or `opts.val` or both must be present. If both are missing, `ast-monkey` will throw an error.

`opts.only` is validated via dedicated package, [util-array-object-or-both](/os/util-array-object-or-both/). Here are the permitted values for `opts.only`, case-insensitive:

| Either type  | Interpreted as array-type | Interpreted as object-type |
| ------------ | ------------------------- | -------------------------- |
| `any`        | `array`                   | `object`                   |
| `all`        | `arrays`                  | `objects`                  |
| `everything` | `arr`                     | `obj`                      |
| `both`       | `aray`                    | `ob`                       |
| `either`     | `arr`                     | `o`                        |
| `each`       | `a`                       |
| `whatever`   |                           |
| `e`          |                           |

If `opts.only` is set to any string longer than zero characters and is not case-insensitively equal to one of the above, the `ast-monkey` will throw an error.

**Output**

The output will be an array, comprising of zero or more plain objects in the following format:

| Object's key | Type             | Description                                                                                                                                                      |
| ------------ | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `index`      | Integer number   | The index of the finding. It's also the last element of the `path` array.                                                                                        |
| `key`        | String           | The found object's key                                                                                                                                           |
| `val`        | Whatever or `null` | The found object's value (or `null` if it's a key of an array)                                                                                                   |
| `path`       | Array            | The found object's path: indexes of all its parents, starting from the topmost. The found key/value pair's address will be the last element of the `path` array. |

If a finding is an element of an array, the `val` will be set to `null`.

**Use example**

Find out, what is the path to the key that equals 'b'.

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = ["a", [["b"], "c"]];
const key = "b";
const result = find(input, { key: key });
console.log("result = " + JSON.stringify(result, null, 4));
// => [
//      {
//        index: 4,
//        key: 'b',
//        val: null,
//        path: [2, 3, 4]
//      }
//    ]
```

Once you know that the path is `[2, 3, 4]`, you can iterate its parents, `get()`-ing indexes number `3` and `2` and perform operations on it. The last element in the findings array is the finding itself.

This method is the most versatile of the `ast-monkey` because you can go "up the AST tree" by querying its array elements backwards.

{% include "btt.njk" %}

### .get()

Use method `get()` to query AST trees by branch's index (a numeric id). You would get that index from a previously performed `find()` or you can pick a number manually.

Practically, `get()` is typically used on each element of the findings array (which you would get after performing `find()`). Then, depending on your needs, you would write the particular index over using `set()` or delete it using `drop()`.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |
| `options`      | Object   | yes         | Options object. See below.                                      |

| Options object | Type                       | Obligatory? | Description                                                           |
| -------------- | -------------------------- | ----------- | --------------------------------------------------------------------- |
| `index`        | Number or number-as-string | yes         | Index number of piece of AST you want the monkey to retrieve for you. |

**Output**

The `get()` returns object, array or `null`, depending what index was matched (or not).

**Use example**

If you know that you want an index number two, you can query it using `get()`:

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = {
  a: {
    b: "c",
  },
};
const index = 2;
const result = get(input, { index: index });
console.log("result = " + JSON.stringify(result, null, 4));
// => {
//      b: 'c'
//    }
```

In practice, you would query a list of indexes programmatically using a `for` loop.

{% include "btt.njk" %}

### .set()

Use method `set()` to overwrite a piece of an AST when you know its index.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |
| `options`      | Object   | yes         | Options object. See below.                                      |

| Options object | Type                       | Obligatory? | Description                                   |
| -------------- | -------------------------- | ----------- | --------------------------------------------- |
| `index`        | Number or number-as-string | yes         | Index of the piece of AST to find and replace |
| `val`          | Whatever                   | yes         | Value to replace the found piece of AST with  |

**Output**

| Output  | Type          | Description         |
| ------- | ------------- | ------------------- |
| `input` | Same as input | The amended `input` |

**Use example**

Let's say you identified the `index` of a piece of AST you want to write over:

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = {
  a: { b: [{ c: { d: "e" } }] },
  f: { g: ["h"] },
};
const index = "7";
const val = "zzz";
const result = set(input, { index: index, val: val });
console.log("result = " + JSON.stringify(result, null, 4));
// => {
//      a: {b: [{c: {d: 'e'}}]},
//      f: {g: 'zzz'}
//    }
```

{% include "btt.njk" %}

### .drop()

Use method `drop()` to delete a piece of an AST with a known index.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |
| `options`      | Object   | yes         | Options object. See below.                                      |

| Options object's key | Type                       | Obligatory? | Description                                                         |
| -------------------- | -------------------------- | ----------- | ------------------------------------------------------------------- |
| `index`              | Number or number-as-string | yes         | Index number of piece of AST you want the monkey to delete for you. |

**Output**

| Output  | Type            | Description         |
| ------- | --------------- | ------------------- |
| `input` | Same as `input` | The amended `input` |

**Use example**

Let's say you want to delete the piece of AST with an index number 8. That's `'h'`:

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = {
  a: { b: [{ c: { d: "e" } }] },
  f: { g: ["h"] },
};
const index = "8"; // can be integer as well
const result = drop(input, { index: index });
console.log("result = " + JSON.stringify(result, null, 4));
// => {
//      a: {b: [{c: {d: 'e'}}]},
//      f: {g: []}
//    }
```

{% include "btt.njk" %}

### .del()

Use method `del()` to delete all chosen key/value pairs from all objects found within an AST, or all chosen elements from all arrays.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |
| `options`      | Object   | yes         | Options object. See below.                                      |

| Options object's key | Type     | Obligatory?                              | Description                                                                                                                                                     |
| -------------------- | -------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `key`                | String   | at least one, `key` or `val`             | All keys in objects or elements in arrays will be selected for deletion                                                                                         |
| `val`                | Whatever | at least one, `key` or `val`             | All object key/value pairs having this value will be selected for deletion                                                                                      |
| `only`               | String   | no (if not given, will default to `any`) | You can specify, to delete key/value pairs (if object) or elements (if array) by setting this key's value to one of the acceptable values from the table below. |

If you set only `key`, any value will be deleted as long as `key` matches. Same with specifying only `val`. If you specify both, both will have to match; otherwise, key/value pair (in objects) will not be deleted. Since arrays won't have any `val`ues, no elements in arrays will be deleted if you set both `key` and `val`.

`opts.only` values are validated via dedicated package, [util-array-object-or-both](/os/util-array-object-or-both/). Here are the permitted values for `opts.only`, case-insensitive:

| Either type  | Interpreted as array-type | Interpreted as object-type |
| ------------ | ------------------------- | -------------------------- |
| `any`        | `array`                   | `object`                   |
| `all`        | `arrays`                  | `objects`                  |
| `everything` | `arr`                     | `obj`                      |
| `both`       | `aray`                    | `ob`                       |
| `either`     | `arr`                     | `o`                        |
| `each`       | `a`                       |
| `whatever`   |                           |
| `e`          |                           |

If `opts.only` is set to any string longer than zero characters and is not case-insensitively equal to one of the above, the `ast-monkey` will throw an error.

**Output**

| Output  | Type            | Description         |
| ------- | --------------- | ------------------- |
| `input` | Same as `input` | The amended `input` |

**Use example**

Let's say you want to delete all key/value pairs from objects that have a key equal to 'c'. Value does not matter.

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = {
  a: { b: [{ c: { d: "e" } }] },
  c: { d: ["h"] },
};
const key = "c";
const result = del(input, { key: key });
console.log("result = " + JSON.stringify(result, null, 4));
// => {
//      a: {b: [{}]}
//    }
```

{% include "btt.njk" %}

### .arrayFirstOnly()

(ex-`flatten()` on versions `v.<3`)

`arrayFirstOnly()` will take an input (whatever), if it's traversable, it will traverse it, leaving only the first element within each array it encounters.

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
const input = [
  {
    a: "a",
  },
  {
    b: "b",
  },
];
const result = arrayFirstOnly(input);
console.log("result = " + JSON.stringify(result, null, 4));
// => [
//      {
//        a: 'a'
//      }
//    ]
```

In practice, it's handy when you want to simplify the data objects. For example, all our email templates have content separated from the template layout. Content sits in `index.json` file. For dev purposes, we want to show, let's say two products in the shopping basket listing. However, in a production build, we want to have only one item, but have it sprinkled with back-end code (loop logic and so on). This means, we have to take data object meant for a dev build, and flatten all arrays in the data, so they contain only the first element. `ast-monkey` comes to help.

**Input**

| Input argument | Type     | Obligatory? | Description                                                     |
| -------------- | -------- | ----------- | --------------------------------------------------------------- |
| `input`        | Whatever | yes         | AST tree, or object or array or whatever. Can be deeply-nested. |

**Output**

| Output  | Type            | Description         |
| ------- | --------------- | ------------------- |
| `input` | Same as `input` | The amended `input` |

{% include "btt.njk" %}

### .traverse()

`traverse()` comes from a standalone library, [ast-monkey-traverse](/os/ast-monkey-traverse/) and you can install and use it as a standalone. Since all methods depend on it, we are exporting it along all other methods. However, it "comes from outside", it's not part of this package's code and the true source of its API is on its own readme. Here, we're just reiterating how to use it.

`traverse()` is an inner method used by other functions. It does the actual traversal of the AST tree (or whatever input you gave, from simplest string to most complex spaghetti of nested arrays and plain objects). This ~method~ function is used via a callback function, similarly to `Array.forEach()`.

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
let ast = [{ a: "a", b: "b" }];
ast = traverse(ast, function (key, val, innerObj) {
  // use key, val, innerObj
  return val !== undefined ? val : key; // (point #1)
});
```

Also, we like to use it this way:

```js
const {
  find,
  get,
  set,
  drop,
  del,
  arrayFirstOnly,
  traverse,
} = require("ast-monkey");
let ast = [{ a: "a", b: "b" }];
ast = traverse(ast, function (key, val, innerObj) {
  let current = val !== undefined ? val : key;
  // All action with variable `current` goes here.
  // It's the same name for any array element or any object key's value.
  return current; // it's obligatory to return it, unless you want to assign that
  // node to undefined
});
```

It's very important to **return the value on the callback function** (point marked `#1` above) because otherwise whatever you return will be written over the current AST piece being iterated.

If you definitely want to delete, return `NaN`.

{% include "btt.njk" %}

#### innerObj in the callback

When you call `traverse()` like this:

```js
input = traverse(input, function (key, val, innerObj) {
  ...
})
```

you get three variables:

- `key`
- `val`
- `innerObj`

If monkey is currently traversing a plain object, going each key/value pair, `key` will be the object's current key and `val` will be the value.
If monkey is currently traversing an array, going through all elements, a `key` will be the current element and `val` will be `null`.

| `innerObj` object's key | Type                                                  | Description                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `depth`                 | Integer number                                        | Zero is root, topmost level. Every level deeper increments `depth` by `1`.                                                                                                                                                                                                                                                                      |
| `path`                  | String                                                | The path to the current value. The path uses exactly the same notation as the popular [object-path](https://www.npmjs.com/package/object-path) package. For example, `a.1.b` would be: input object's key `a` > value is array, take `1`st index (second element in a row, since indexes start from zero) > value is object, take it's key `b`. |
| `topmostKey`            | String                                                | When you are very deep, this is the topmost parent's key.                                                                                                                                                                                                                                                                                       |
| `parent`                | Type of the parent of current element being traversed | A whole parent (array or a plain object) which contains the current element. Its purpose is to allow you to query the **siblings** of the current element.                                                                                                                                                                                      |

{% include "btt.njk" %}

## The name of this library

HTML is parsed into nested objects and arrays which are called Abstract Syntax Trees. This library can go up and down the trees, so what's a better name than _monkey_? The **ast-monkey**. Anything-nested is can also be considered a tree – tree of plain objects, arrays and strings, for example. Monkey can [traverse](#traverse) anything really.

{% include "btt.njk" %}
