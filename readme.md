# fast-redact

very fast object redaction

[![Build Status](https://travis-ci.org/davidmarkclements/fast-redact.svg?branch=master)](https://travis-ci.org/davidmarkclements/fast-redact)

## Default Usage

By default, `fast-redact` serializes an object with `JSON.stringify`, censoring any 
data at paths specified:

```js
const fastRedact = require('fast-redact')
const fauxRequest = {
  headers: {
    host: 'http://example.com',
    cookie: `oh oh we don't want this exposed in logs in etc.`,
    referer: `if we're cool maybe we'll even redact this`
  }
}
const redact = fastRedact({
  paths: ['headers.cookie', 'headers.referer']
})

console.log(redact(fauxRequest))
// {"headers":{"host":"http://example.com","cookie":"[REDACTED]","referer":"[REDACTED]"}}
```

## API

### `require('fast-redact')({paths, censor, serialize}) => Function`

When called without any options, or with a zero length `paths` array, 
`fast-redact` will return  `JSON.stringify` or the `serialize` option, if set.

#### `paths` – `Array`

An array of strings describing the nested location of a key in an object.

The syntax follows that of the EcmaScript specification, that is any JavaScript
path is accepted – both bracket and dot notation is supported. For instance in 
each of the following cases, the `c` property will be redacted: `a.b.c`,`a['b'].c`, 
`a["b"].c`, `a[``b``].c`. Since bracket notation is supported, array indices are also
supported `a[0].b` would redact the `b` key in the first object of the `a` array. 

##### Wildcards

In addition to static paths, asterisk wildcards are also supported.

When an asterisk is place in the final position it will redact all keys within the
parent object. For instance `a.b.*` will redact all keys in the `b` object. Similarly
for arrays `a.b[*]` will redact all elements of an array (in truth it actually doesn't matter 
whether `b` is in an object or array in either case, both notation styles will work).

When an asterisk is in an intermediate or first position, the paths following the asterisk will 
be redacted for every object within the parent.

For example:

```js
const fastRedact = require('fast-redact')
const redact = fastRedact({paths: ['*.c.d']})
const obj = {
  x: {c: {d: 'hide me', e: 'leave me be'}},
  y: {c: {d: 'and me', f: 'I want to live'}},
  z: {c: {d: 'and also I', g: 'I want to run in a stream'}}
}
console.log(redact(obj)) 
// {"x":{"c":{"d":"[REDACTED]","e":"leave me be"}},"y":{"c":{"d":"[REDACTED]","f":"I want to live"}},"z":{"c":{"d":"[REDACTED]","g":"I want to run in a stream"}}}
```

Another example with a nested array:

```js
const fastRedact = require('..')
const redact = fastRedact({paths: ['a[*].c.d']})
const obj = {
  a: [
    {c: {d: 'hide me', e: 'leave me be'}},
    {c: {d: 'and me', f: 'I want to live'}},
    {c: {d: 'and also I', g: 'I want to run in a stream'}}
  ]
}
console.log(redact(obj)) 
// {"a":[{"c":{"d":"[REDACTED]","e":"leave me be"}},{"c":{"d":"[REDACTED]","f":"I want to live"}},{"c":{"d":"[REDACTED]","g":"I want to run in a stream"}}]}
```

#### `remove` - `Boolean` - `[false]`

The `remove` option, when set to `true` will cause keys to be removed from the 
serialized output. 

Since the implementation exploits the fact that `undefined` keys are ignored
by `JSON.stringify` the `remove` option may *only* be used when `JSON.stringify`
is the serializer (this is the default) – otherwise `fast-redact` will throw. 

If supplying a custom serializer that has the same behavior (removing keys 
with `undefined` values), this restriction can be bypassed by explicitly setting 
the `censor` to `undefined`.


#### `censor` – `<Any type - Except Function>` – `('[REDACTED]')`

This is the value which overwrites redacted properties. It can be any type, 
except a function. 

Setting `censor` to `undefined` will cause properties to removed as long as this is 
the behavior of the `serializer` – which defaults to `JSON.stringify`, which does 
remove `undefined` properties.

Supplying a function as a `censor` will cause `fast-redact` to throw. This means 
a function censor to be introduced in the future without it being a breaking change. 

#### `serialize` – `Function | Boolean` – `(JSON.stringify)`

The `serialize` option may be a function of a boolean. If a function is supplied, this
will be used to `serialize` the redacted object. It's important to understand that for 
performance reasons `fast-redact` *mutates* the original object, then serializes, then 
restores the original values. So the object passed to the serializer is the exact same
object passed to the redacting function. 

The `serialize` option as a function example:

```js
const fastRedact = require('fast-redact')
const redact = fastRedact({
  paths: ['a'], 
  serialize: (o) => JSON.stringify(o, 0, 2)
})
console.log(redact({a: 1, b: 2}))
// {
//   "a": "[REDACTED]",
//   "b": 2
// }
```

For advanced usage the `serialize` option can be set to `false`. When `serialize` is set to `false`,
instead of the serialized object, the output of the redactor function will be the mutated object 
itself (this is the exact same as the object passed in). In addition a `restore` method is supplied
on the redactor function allowing the redacted keys to be restored with the original data. 

```js
const fastRedact = require('fast-redact')
const redact = fastRedact({
  paths: ['a'], 
  serialize: false
})
const o = {a: 1, b: 2}
console.log(redact(o) === o) // true
console.log(o) // { a: '[REDACTED]', b: 2 }
console.log(redact.restore(o) === o) // true
console.log(o) // { a: 1, b: 2 }
```

## Approach

In order to achieve lowest cost/highest performance redaction `fast-redact`
creates and compiles a function (using the `Function` constructor) on initialization.
It's important to distinguish this from the dangers of a runtime eval, no user input 
is involved in creating the string that compiles into the function. This is as safe 
as writing code normally and having it compiled by V8 in the usual way.

Thanks to changes in V8 in recent years, state can be injected into compiled functions
using `bind` at very low cost (whereas `bind` used to be expensive, and getting state
into a compiled function by any means was difficult without a performance penalty).

For static paths, this function simply checks that the path exists and then overwrites
with the censor. Wildcard paths are processed with normal functions that iterate over 
the object redacting values as necessary.

It's important to note, that the original object is mutated – for performance reasons
a copy is not made. See [rfdc](https://github.com/davidmarkclements/rfdc) (Really Fast 
Deep Clone) for the fastest known way to clone – it's not nearly close enough  in speed
to editing the original object, serializing and then restoring values. 

A `restore` function is also created and compiled to put the original state back on
to the object after redaction. This means that in the default usage case, the operation 
is essentially atomic - the object is mutated, serialized and restored internally which 
avoids any state management issues.

## Caveat

As mentioned in approach, the `paths` array input is dynamically compiled into a function
at initialization time. While the `paths` array is vigourously tested for any developer 
errors, it's strongly recommended against allowing user input to directly supply any 
paths to redact. It can't be guaranteed that allowing user input for `paths` couldn't
feasibly expose an attack vector.  

## Benchmarks

The fastest known predecessor to `fast-redact` is the non-generic [`pino-noir`](http://npm.im/pino-noir) 
library (which was also written by myself).

In the direct calling case, `fast-redact` is over 25x faster than `pino-noir`, however a more realistic
comparison is overhead on `JSON.stringify`. 

For a static redaction case (no wildcards) `pino-noir` adds ~28% overhead on top of `JSON.stringify`
whereas `fast-redact` adds ~2% overhead.

In the basic last-position wildcard case,`fast-redact` is ~5% faster than `pino-noir`, and 
intermediate wildcard redaction (which `pino-noir` does not support) is still faster than 
the `pino-noir` basic wildcard case. 

```sh
npm run bench 
```

```
benchNoirV2*500: 52.387ms
benchFastRedact*500: 2.361ms
benchFastRedactRestore*500: 13.715ms
benchNoirV2Wild*500: 75.820ms
benchFastRedactWild*500: 29.813ms
benchFastRedactWildRestore*500: 36.182ms
benchFastRedactIntermediateWild*500: 185.323ms
benchFastRedactIntermediateWildRestore*500: 73.520ms
benchJSONStringify*500: 268.861ms
benchNoirV2Serialize*500: 372.731ms
benchFastRedactSerialize*500: 273.265ms
benchNoirV2WildSerialize*500: 330.679ms
benchFastRedactWildSerialize*500: 312.374ms
benchFastRedactIntermediateWildSerialize*500: 325.921ms
```

## Tests

```
npm test  
```

```
  194 passing (450.351ms)
```

### Coverage

```
npm run cov 
```

```
-----------------|----------|----------|----------|----------|-------------------|
File             |  % Stmts | % Branch |  % Funcs |  % Lines | Uncovered Line #s |
-----------------|----------|----------|----------|----------|-------------------|
All files        |      100 |      100 |      100 |      100 |                   |
 fast-redact     |      100 |      100 |      100 |      100 |                   |
  index.js       |      100 |      100 |      100 |      100 |                   |
 fast-redact/lib |      100 |      100 |      100 |      100 |                   |
  modifiers.js   |      100 |      100 |      100 |      100 |                   |
  parse.js       |      100 |      100 |      100 |      100 |                   |
  redactor.js    |      100 |      100 |      100 |      100 |                   |
  restorer.js    |      100 |      100 |      100 |      100 |                   |
  rx.js          |      100 |      100 |      100 |      100 |                   |
  state.js       |      100 |      100 |      100 |      100 |                   |
  validator.js   |      100 |      100 |      100 |      100 |                   |
-----------------|----------|----------|----------|----------|-------------------|
```

## License

MIT

## Acknowledgements

Sponsored by [nearForm](http://www.nearform.com)