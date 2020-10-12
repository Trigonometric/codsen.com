---
layout: package
title: generate-atomic-css
packages:
  - email-comb
---

## API

Three methods are exported:

```js
const {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} = require("generate-atomic-css");
// or
import {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} from "generate-atomic-css";
```

| Name                | What it does                                                     |
| ------------------- | ---------------------------------------------------------------- |
| genAtomic           | It's the main function which generates CSS                       |
| version             | Exports version from package.json, for example, string `1.0.1`   |
| headsAndTails       | Exports a plain object with all heads and tails                  |
| extractFromToSource | Extracts "from" and "to" from source in rows, separated by pipes |

{% include "btt.njk" %}

## genAtomic() - input API

It's a function, takes two arguments: input string and optional options object:

**genAtomic(str, [originalOpts])**

{% include "btt.njk" %}

### genAtomic() - Optional Options Object

It's a plain object which goes into second input argument of the main function, `genAtomic()`.
Here are all the keys and their values:

| Options Object's key     | The type of its value    | Default | Description                                                                                                                                                                                      |
| ------------------------ | ------------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `includeConfig`          | boolean                  | `true`  | Should config be repeated, wraped with `GENERATE-ATOMIC-CSS-CONFIG-STARTS` and `GENERATE-ATOMIC-CSS-CONFIG-ENDS`? Enabling this enables `includeHeadsAndTails` as well (if not enabled already). |
| `includeHeadsAndTails`   | boolean                  | `true`  | Should the generated CSS be wrapped with `GENERATE-ATOMIC-CSS-CONFIG-STARTS` and `GENERATE-ATOMIC-CSS-CONFIG-ENDS`?                                                                              |
| `pad`                    | boolean                  | `true`  | Should the numbers be padded                                                                                                                                                                     |
| `configOverride`         | `null` (off) or `string` | `null`  | This is override, you can hard-set the config from outside. Handy when input contains old/wrong config.                                                                                          |
| `reportProgressFunc`     | function or `null`         | `null`  | Handy in worker setups, if you provide a function, it will be called for each percentage done from `reportProgressFuncFrom` to `reportProgressFuncTo`, then finally, with the result.            |
| `reportProgressFuncFrom` | natural number           | `0`     | `reportProgressFunc()` will ping unique percentage progress once per each percent, from 0 to 100 (%). You can skew the starting percentage so counting starts not from zero but from this.       |
| `reportProgressFuncTo`   | natural number           | `100`   | `reportProgressFunc()` will ping unique percentage progress once per each percent, from 0 to 100 (%). You can skew the starting percentage so counting starts not from zero but from this.       |

Here it is, in one place, in case you want to copy-paste it somewhere:

```js
{
  includeConfig: true,
  includeHeadsAndTails: true,
  pad: true,
  configOverride: null,
  reportProgressFunc: null,
  reportProgressFuncFrom: 0,
  reportProgressFuncTo: 100
};
```

{% include "btt.njk" %}

## genAtomic() - output API

The main function genAtomic() exports a plain object where result string is under key `result`:

For example, here's the format of the main function's output:

```js
{
  log: {
    count: 1000
  },
  result: "<bunch of generated CSS>"
}
```

You tap the result like this:

```js
import {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} from "generate-atomic-css";
const source = `.mt$$$ { margin-top: $$$px; }|3`;
const { log, result } = genAtomic(source, {
  includeConfig: true,
});
console.log(`total generated classes and id's: ${log.count}`);
// => 4
console.log(`result:\n${result}`);
// => <bunch of generated CSS>
```

{% include "btt.njk" %}

## version

It's a string and it comes straight from `package.json`. For example:

```js
import {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} from "generate-atomic-css";
console.log(`version = v${version}`);
// => version = v1.0.1
```

{% include "btt.njk" %}

## headsAndTails

It's a plain object, its main purpose is to serve as a single source of truth for heads and tails names:

```js
{
  CONFIGHEAD: "GENERATE-ATOMIC-CSS-CONFIG-STARTS",
  CONFIGTAIL: "GENERATE-ATOMIC-CSS-CONFIG-ENDS",
  CONTENTHEAD: "GENERATE-ATOMIC-CSS-CONTENT-STARTS",
  CONTENTTAIL: "GENERATE-ATOMIC-CSS-CONTENT-ENDS"
}
```

For example,

```js
import {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} from "generate-atomic-css";
console.log(`headsAndTails.CONTENTTAIL = ${headsAndTails.CONTENTTAIL}`);
// => headsAndTails.CONTENTTAIL = GENERATE-ATOMIC-CSS-CONTENT-ENDS
```

{% include "btt.njk" %}

## extractFromToSource()

It's an internal function which reads the source line, for example:

```js
.pb$$$ { padding-bottom: $$$px !important; } | 5 | 10
```

and separates "from" (`5` above) and "to" (`10` above) values from the rest of the string (`.pb$$$ { padding-bottom: $$$px !important; }`).

The challenging part is that pipes can be wrapping the line from outside, plus, if there is only one number at the end of the line, it is "to" value.

```
| .mt$$$ { margin-top: $$$px !important; } | 1 |
```

Here's an example how to use `extractFromToSource()`:

```js
const {
  genAtomic,
  version,
  headsAndTails,
  extractFromToSource,
} = require("generate-atomic-css");
const input1 = `.pb$$$ { padding-bottom: $$$px !important; } | 5 | 10`;
const input2 = `.mt$$$ { margin-top: $$$px !important; } | 1`;

// second and third input argument are default "from" and default "to" values:
const [from1, to1, source1] = extractFromToSource(input1, 0, 500);
console.log(`from = ${from1}`);
// from = 5
console.log(`to = ${to1}`);
// from = 10
console.log(`source = "${source1}"`);
// source = ".pb$$$ { padding-bottom: $$$px !important; }"

const [from2, to2, source2] = extractFromToSource(input2, 0, 100);
console.log(`from = ${from2}`);
// from = 0 <--- default
console.log(`to = ${to2}`);
// from = 1 <--- comes from pipe, "} | 1`;"
console.log(`source = "${source2}"`);
// source = ".mt$$$ { margin-top: $$$px !important; }"
```

{% include "btt.njk" %}

## Idea

On a basic level, you can turn off heads/tails (set `opts.includeHeadsAndTails` to `false`) and config (set `opts.includeConfig` to `false`).

Each line which contains `$$$` will be repeated, from default `0` to `500` or within the range you set:

```css
.pb$$$ { padding-bottom: $$$px !important; } | 5 | 10
```

Above instruction means generate from `5` to `10`, inclusive:

```css
.pb5 {
  padding-bottom: 5px !important;
}
.pb6 {
  padding-bottom: 6px !important;
}
.pb7 {
  padding-bottom: 7px !important;
}
.pb8 {
  padding-bottom: 8px !important;
}
.pb9 {
  padding-bottom: 9px !important;
}
.pb10 {
  padding-bottom: 10px !important;
}
```

If you're happy to start from zero, you can put only one argument, "to" value:

```css
.w$$$p { width: $$$% !important; } | 100
```

Above instruction means generate from (default) `0` to (custom) `100`, inclusive:

```css
/* GENERATE-ATOMIC-CSS-CONTENT-STARTS */
.w0p {
  width: 0 !important;
}
.w1p {
  width: 1% !important;
}
.w2p {
  width: 2% !important;
}
.... .w98p {
  width: 98% !important;
}
.w99p {
  width: 99% !important;
}
.w100p {
  width: 100% !important;
}
```

{% include "btt.njk" %}

## Config

What happens if you want to edit the generated list, to change ranges, to add or remove rules?

You need to recreate the original "recipe", lines `.pb$$$ { padding-bottom: $$$px !important; }` and so on.

Here's where the config comes to help.

Observe:

```css
/* GENERATE-ATOMIC-CSS-CONFIG-STARTS
.pb$$$ { padding-bottom: $$$px !important; } | 5 | 10

.mt$$$ { margin-top: $$$px !important; } | 1
GENERATE-ATOMIC-CSS-CONFIG-ENDS
GENERATE-ATOMIC-CSS-CONTENT-STARTS */
.pb5 {
  padding-bottom: 5px !important;
}
.pb6 {
  padding-bottom: 6px !important;
}
.pb7 {
  padding-bottom: 7px !important;
}
.pb8 {
  padding-bottom: 8px !important;
}
.pb9 {
  padding-bottom: 9px !important;
}
.pb10 {
  padding-bottom: 10px !important;
}

.mt0 {
  margin-top: 0 !important;
}
.mt1 {
  margin-top: 1px !important;
}
/* GENERATE-ATOMIC-CSS-CONTENT-ENDS */
```

If `opts.includeConfig` setting is on (it's on by default), your original config will be placed on top of generated content.

Furthermore, if generator detects content heads and tails placeholders, it will wipe existing contents there, replacing them with newly generated CSS.

The idea is you should be able to keep your config in your master email template, only remove config like regular CSS comment when deploying to production. But you'd still keep the master template with config. Later you could reuse it.

{% include "btt.njk" %}
