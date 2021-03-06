---
layout: package
title: test-mixer
packages:
  - object-boolean-combinations
---

## Purpose

It's used to generate an array of all possible combinations of options object boolean settings.

[`detergent`](/os/detergent/) has 12 boolean toggles — that's `2^12 = 4096` variations to test for each input. This program generates those options variations.

[`string-collapse-white-space`](/os/string-collapse-white-space/) has 6 boolean toggles — that's `2^6 = 64` variations to test for each input.

And so on.

{% include "btt.njk" %}

## API - Input

::: api
mixer(
  objWithFixedKeys,
  [defaultsObj]
)
:::

In other words, it's a function which takes two input arguments, second one being optional (marked by square brackets).

| Input argument   | Type         | Obligatory? | Description                                                                                                                           |
| ---------------- | ------------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `objWithFixedKeys`    | Plain object | yes, but can be empty         | Object with boolean keys that should not be used to generate variations. |
| `defaultsObj` | Plain object | yes, can be empty, should be a superset of `objWithFixedKeys` | Put the default options object here. |

{% include "btt.njk" %}

## API - Output

Program returns an array of zero or more plain objects.

{% include "btt.njk" %}

## In practice

Let's say, we test [detergent](/os/detergent). It has the following default opts [`defaultOpts` in the source](https://github.com/codsen/codsen/blob/main/packages/detergent/src/util.js):

```js
{
  fixBrokenEntities: true,
  removeWidows: true,
  convertEntities: true,
  convertDashes: true,
  convertApostrophes: true,
  replaceLineBreaks: true,
  removeLineBreaks: false,
  useXHTML: true,
  dontEncodeNonLatin: true,
  addMissingSpaces: true,
  convertDotsToEllipsis: true,
  stripHtml: true,
  eol: "lf",
  stripHtmlButIgnoreTags: ["b", "strong", "i", "em", "br", "sup"],
  stripHtmlAddNewLine: ["li", "/ul"],
  cb: null,
}
```

Imagine, we're testing how detergent strips HTML, setting, `opts.stripHtml`.

```js
// 😱 eleven other boolean opts settings were left out, 2^12 - 1 = 4095 other combinations!
import tap from "tap";
const { det } = require("detergent");
tap.test(`01`, (t) => {
  t.equal(
    det(t, n, `text <a>text</a> text`, {
      stripHtml: true,
    }).res,
    "text text text"
  );
  t.end();
});
```

Here's the proper way:

```js
// ✅
import tap from "tap";
const { det, opts } = require("detergent");
import { mixer as testMixer } from "text-mixer";

// we create a wrapper to skip writing the second input argument again and again
const mixer = (ref) => testMixer(ref, opts);

tap.test(`01`, (t) => {
  // 2^11 = 2048 variations:
  mixer({
    stripHtml: false, // <--- will be the same across all generated objects
  }).forEach((opt, n) => {
    t.equal(
      det(t, n, `text <a>text</a> text`, opt).res,
      `text <a>text</a> text`,
      JSON.stringify(opt, null, 4)
    );
  });
  // another 2^11 = 2048 variations:
  mixer({
    stripHtml: true,
  }).forEach((opt, n) => {
    t.equal(
      det(t, n, `text <a>text</a> text`, opt).res,
      `text text text`,
      JSON.stringify(opt, null, 4)
    );
  });
  t.end();
});
```

{% include "btt.njk" %}
