# Algorithms

## Resolving a module path
Module resolution would be identical to how it proceeds today. There is no change in file extensions
to look for, and no module path mapping to perform. The only change being if no `main` field exists
in a `package.json` file, then it will attempt to find a `module` field value for the package entry point.

## Determining the module format
Effectively, the module format defaults to CommonJS, unless the `package.json` has a `module` instead 
of a `main` property, in which case the default is `ES2015`. Even then, globbing patterns may still
set a default format of CommonJS for certain module paths. 

1. Let `module_path` be the full path of the resolved module. 
2. Let `module_format` equal `CommonJS`.
3. If the module is located in a package with a `package.json` file
  1. If the `package.json` file contains a `main` field, exit to step 4.
  2. If the `package.json` file does not contain a `module` field, exit to step 4.
  3. Let `module_format` equal `ES2015`.
  4. If the `package.json` file does not contain a `commonjs_modules` property with an array value, exit to step 4.
  5. Let `module_relative_path` be the value of `module_path` relative to the `package.json` directory.
  6. For each element in the array value for `commonjs_modules`:
    1. Let `glob_value` be the element of the array.
    2. If `typeof glob_value !== "string"` then continue to the next element.
    3. If `module_relative_path` matches the value of `glob_value` when treated as a globbing pattern, let `module_format` equal `CommonJS`.
4. Attempt to parse the file at `module_path` as a module of format `module_format`.
5. If a parse error occurs, attempt to parse the file with the alternate format.

Per steps 4 and 5, if a parsing error occurs, the alternate format will be attempted. This is to provide for the ability
to use ES2015 modules even when no configuration information is available (e.g. no `package.json`), or 
is not simply (correctly) configured. This is with the expectation that ES2015 modules will contain `import`
or `export` statements, which will fail to parse as the non-module goal used for parsing CommonJS modules.

The globbing allows for a gradual migration of a package. For example, the below would specify that 
this package contains ES2015 modules, expect for the modules in the `foo` and `widget` directories, which 
are still in CommonJS format:

```json
{
  "name": "big-app",
  "version": "2.0.0",
  "module": "index.js",
  "commonjs_modules": ["**/foo/*.js", "**/widget/*.js"]
}
```

Once migration to ES2015 modules is complete, the `commonjs_modules` field is simply removed.

## Loading CommonJS modules via ES2015 import statements
Effectively, CommonJS modules appear as the `default` member of an ES2015 module to allow for
semantics and coding patterns similar to CommonJS.

1. Let `name` equal the value of the `from` clause of an import statement.
2. Let `cjs_module` equal the value as would be return from a `require(name)` call in Node.js.
3. Let `result` equal the object value `{"default": cjs_module}`
4. Return `result` as the object representing the ES2015 module.

Projecting the CommonJS module as the ES2015 `default` members allows for semantics that would
not otherwise be permitted, such as the module be callable or newable, or having settable properties.
For example:

```javascript
import EventEmitter from "events";
// The "events" module is Node.js is both newable and exposes a setting, e.g.
class MyEmitter extends EventEmitter {
  // TODO
}
EventEmitter.defaultMaxListeners = 100;
```

## Loading ES2015 modules via CommonJS "require" calls
The only special handling for ES2015 modules is that if they only export a `default` member, then
this member should be `hoisted` to become the module value. This allows for simulating a callable
`module.exports`, and for easier migrating and round-tripping of ES2015 and CommonJS modules.

1. Let `name` equal the first argument to the `require` call.
2. Let `es_module` equal the value as would be assign to `ns` in the statement: `import * as ns from "name"`.
3. If `es_module` only contains a `default` property, then let `es_module` equal the `default` property value.
4. Return `es_module` as the result of the `require` call.

See below example for converting the Node.js "assert" module for when this is needed.

## Member hoisting
In order to facilitate migrating existing CommonJS code to a more ES2015-style coding pattern, additional 
hoisting could be provided.

### When loading CommonJS modules via ES2015 imports
In the algorithm already outlined, after step 3:
- For each own property on `csj_module`
  1. Let `prop` equal the name of the property. (If the name has the value "default", skip this property).  
  2. Add a getter named `prop` on `result` that returns the value of `cjs_module[prop]`.

This allows for code such as the below to work with existing CommonJS modules:

```javascript
import {statSync, readFile} from "fs"
```

### When loading ES2015 modules as ES2015 modules
If the ES2015 module only has a `default` export, then hoist the own properties on `default` to the module itself:

1. Let `module_value` equal the result of evaluating the imported module.
2. If `module_value` only contains a `default` member, then for each property on `default`:
  1. Let `prop_name` equal the name of the property.
  2. Add a getter to `module_value` with the name `prop_name` that return the value of `default[prop_name]`.

This allow for converting code via the process as outlined below:

```javascript

// Existing CommonJS module
function assert(test, msg){ /* TODO */ }
assert.ok = function (msg) { /* TODO */ }
assert.fail = function (msg) { /* TODO */ }
assert.deepEquals = function (actual, expected) { /* TODO */ }
// etc...
module.exports = assert;


// Converted to a ES2015 module that appears identical when loaded via "require" calls, would be written as: 
function assert(test, msg){ /* TODO */ }
assert.ok = function (msg) { /* TODO */ }
assert.fail = function (msg) { /* TODO */ }
assert.deepEquals = function (actual, expected) { /* TODO */ }
// etc...
export default assert;


// Without hoisting the above in ES2015 imports, still requires CommonJS-like usage in ES2015 imports
import assert from "assert";
assert.ok("test");
// etc...


// With hoisting in ES2015 imports also, can be used with named import list pattern
import {ok, deepEquals} from "assert";
ok("test");
// etc...
```
