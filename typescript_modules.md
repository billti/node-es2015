# Node and ES2015 modules in TypeScript

## Code generation
Today, in TypeScript, you might write code to use Node.js modules such as the below:

```typescript
import * as assert from "assert";           // Importing a CommonJS module as a namespace
import fs = require('fs');                  // Importing using ** TypeScript specific syntax **

var stats = fs.statSync("/temp/file.txt");  
assert(stats.isFile(), "File missing");     // ** Invalid invoking of the namespace as a function **
```

There are a couple of issues with the above, which are highlighted in the comment.

 - While the first namespace import is valid ES2015 syntax, it _should_ result in an identifier
 which is not callable. However for compatiblity, currently this is not the case.
 - The second import is TypeScript specific syntax that addresses the above. The resulting 
 identifier is equivalent to the CommonJS/JavaScript code `var fs = require("fs")`.

Both approaches have problems; one is invalid semantics, and the other invalid syntax. For 
reference, it results in the below ES5 code:

```javascript
var assert = require("assert");
var fs = require('fs');
var stats = fs.statSync("/temp/file.txt");
assert(stats.isFile(), "File missing");
```

Identifiers may be imported directly from a module, this TypeScript code looks like:

```typescript
import {statSync, readFile} from "fs";

var stats = statSync("/temp/file.txt");
if (stats.isFile()) {
    readFile("/temp/file.txt", file => {/* TODO */})
}
```

Which results in the below emitted ES5 code, which is similar in appearance to the namespace code:

```javascript
var fs_1 = require("fs");

var stats = fs_1.statSync("/temp/file.txt");
if (stats.isFile()) {
    fs_1.readFile("/temp/file.txt", function (file) { });
}
```

The challenge with using namespaces or named imports, is that this does not accurately reflect
ES2015 semantics. In ES2015, as well as the module not being callable (as outlined above), its
exports are not mutable.

Using the `default` property avoids these problems, as it can be both callable and mutable. This
does however require that the CommonJS module be _hoisted_ to this property. The TypeScript type
system supports this today (with `allowSyntheticDefaultImports` set to `true`), and the resulting
code would be written something like:

```typescript
import assert from "assert";
import fs from "fs";

var stats = fs.statSync("/temp/file.txt");
assert(stats.isFile());
fs.readFile("/temp/file.txt", file => {/* TODO */});
```

However the synthetic `default` property only exists in the type system, the emitted code does
not do any _hoisting_, which means the resulting code is invalid - as there is no `default` at
runtime:

```javascript
var assert_1 = require("assert");
var fs_1 = require("fs");
var stats = fs_1["default"].statSync("/temp/file.txt");
assert_1["default"](stats.isFile());
fs_1["default"].readFile("/temp/file.txt", function (file) { });
```

What would be needed is a setting such as `useSyntheticDefaultImports` which would effectively
behave as if the CommonJS module exists as the `default` property of the imported module. Thus the
emitted code would be as you would typically write the CommonJS code by hand, e.g.

```javascript
var assert = require("assert");
var fs = require("fs");
var stats = fs.statSync("/temp/file.txt");
assert(stats.isFile());
fs.readFile("/temp/file.txt", function (file) { });
```

This is the proposal: That effectively every CommonJS module `"foo"`, would just appear to ES2015 imports
as the `default` property on the ES2015 module `"foo"`.

This does mean you cannot use an import list with CommonJS modules, but it also means the code you
write in ES2015 syntax closely mirrors existing code (e.g. just replace `var fs = require("fs")`
with `import fs from "fs"`, and if desired, you can use destructuring in ES2015 to extract the
identifiers you want in scope for the module, e.g.

```typescript
import fs from "fs";
const {statSync, readFile} = fs;
// Can use statSync and readFile directly in the rest of the module.
```

Note: Should the additional hoisting outlined in the `algorithms.md` document also eventuate, then
using named imports would also work.

## Type definitions
For TypeScript, the resulting challenge is how to model modules in this environment. How do you
distinguish between a type definition for a CommonJS module (which should act as if it exists on the
`default` property), and an ES2015 module (which should allow for an import list).

For example, if module "fs" became an ES2015 module, then the _exports_ would be described the same,
however the code could be written as shown above, i.e. `import {statSync, readFile} from "fs";`

Today in TypeScript, a CommonJS module is often signified with `export =`, as this is the only way
to have a call signature on a module (which again, is not allowed in ES2015). A sample `d.ts` file 
for TypeScript 2.0 for a CommonJS module might appear as:

```typescript
import * as Express from 'express';

declare function auth(req: Express.Request): auth.BasicAuthResult;

declare namespace auth {
    interface BasicAuthResult {
        name: string;
        pass: string;
    }
}

export = auth;
```

The proposal is this: When emitting code to target ES2015 module syntax, a module that uses
an `export = ...` syntax to define itself, is considered a "legacy" CommonJS module, and thus its
value is available on the `default` property (i.e. is equivalent to have written `export default ...`).

Native ES2015 modules should be written as (and have their .d.ts files generated as) top level 
"export" members, not use the `export = ` syntax. (This is already the recommended approach).

# Compatibility
This has no backwards compatibility issue, as there are no ES2015 module packages in Node.js
today, and thus no code that targets ES2015 module syntax when generating Node.js code. Existing
code generated for Node.js today (via `"moduleKind": "commonjs"`) will
continue to work as it does today.

# Todo
Further examples of the matrix of use cases, e.g.
 - Existing code using the new module kind.
 - New code using the old module kind.
 - New code using the new module kind.
 - The above for ES2015 and ES5 emit.
 - The above for consuming CommonJS and ES2015 modules.
 - Migration scenarios as code (consumer and consumed) moves from CommonJS to ES2015.
 - Note any other possible scenarios and if they are valid.
 - Consider codebases that DO target ES2015 module syntax for Node.js today and then use
 another transpiler (e.g. Babel).
