# ES2015 modules in Node.js

## The problem
Node.js is an amazing product which is at a critical point. The growth of the 
Node.js ecosystem has been phenominal, with over a billion packages a week now 
installed via NPM. A large part of this success has been the Node.js module system,
and the simple to use CommonJS module format, which provides a much needed way to
isolate and distribute pieces of JavaScript functionality, without the "global
namespace" pollution that is well known to many JavaScript developers.

However, with the ES2015 version of the JavaScript standard, the language now provides
a way to import and export modules, and this does not interoperate seemlessly with
the CommonJS design. Without resolving this tension, there is a risk of fracturing
the JavaScript ecosystem, with Node.js developers continuing to write modules in
the CommonJS format, and other JavaScript developers writing to the ES2015 standard,
reducing the ability to share code, tools, and skills across development efforts.

Potentially worse than not solving the problem, is implementing a clumsy solution,
making the "JavaScript modules" space more confusing than it already is, and introducing
further subtlety and nuance that makes learning and writing correct code even more
difficult. This would discourage new developers and add cost for existing ones, as
well as making tooling more complex, and bugs harder to fix.

## The need
Even before the ES2015 standard was finalized, the newer features of the language
saw rapid adoption. This was enabled via tools such as Babel, TypeScript, SystemJS,
etc., which provided the ability to write code using the latest features, and generate
code that would work in the engines of the day. As no engines supported ES2015
modules when the spec was finalized, all these tools generated code to target the
module systems that did exist (e.g. CommonJS and RequireJS), and thus interoperability
issues were relatively limited. 

Today however, all the major JavaScript engines have started to implement ES2015
module support, and already support most of the other features of the ES2015
standard. Thus developers have a desire to write "standard JavaScript" and have
it run in the latest engines, without the need to significantly modify the code.
Node.js developers have typically been keen to see the platform move forward
rapidly and not become stale (e.g. see "io.js").

Without ES2015 modules, it is possible (and common) to take a piece of JavaScript, 
and wrap it in a structure that detects the current environments (i.e. global script, 
RequireJS module, or Node.js) and run accordingly. ES2015 modules are designed to 
ease this problem, and provide a common module format in the language. As browsers 
add support for ES2015 modules, more code will start to be written using this format.
If Node.js does not support this format, then such code will not be reusable in a 
Node.js environment. (Especially as the new ES2015 syntax is a parse error in runtimes
that do not understand it).

## The goal
The primary goal is that CommonJS modules need to continue to work as-is - there 
is too large and too valuable an ecosystem to re-write or abandon, or even introduce
minor breaking change to. These need to be consumable from new and existing CommonJS 
modules without changes, as well as usable from ES2015 modules while providing identical
behavior.

The ability to write and execute modules written with the ES2015 module syntax
is also paramount. This enables the entire JavaScript ecosystem to move forward,
share code, and use tools with features from the latest JavaScript standard. Ideally,
if developers wish to continue writing code using the CommonJS style, they should
also be able to consume ES2015 modules (assuming they are using a Node.js runtime
that supports loading them).

Existing code and tools should be reusable where possible, or simple to migrate
where not possible. Examples include transpilers such as Babel and TypeScript, loaders
such as SystemJS, module definitions such as Definitely Typed, etc.

Behavior should be simple and consistent, without the need for historical knowledge
of module system nuances to understand code, or the need for complex handling for 
subtle changes in behavior depending on the runtime version or module format being used.
One of _the most_ compelling features of Node.js has been how easy it is to learn
and use. If this is compromised, the platform suffers.  

## The challenges
A number of challenges exist in arriving at an elegant and simple solution for
having CommonJS and ES2015 modules coexist in Node.js. Briefly, these fall into
two main categories.

### How to determine what format a module is
The below challenges exist for knowing whether to treat a module as a CommonJS or
a ES2015 module.

#### Advance knowledge of the module format
CommonJS modules and ES2015 modules have different parse goals, which have different
semantics (most notably, ES2015 modules are implicitly in "strict mode"), thus they
need to be parsed differently. Besides being impractical to try both for performance
reasons, they are also ambiguous anyway (specifically, a ES2015 module with no imports
or exports, of which there are examples, will parse correctly as a CommonJS module).

#### Gradual migration
It is not practical to require that all modules in a package or application be of
one format. There are many large apps that will take significant effort to migrate,
and thus will need to exist in a state of mixed CommonJS and ES2015 modules as migration
occurs. Thus related to the above, the loader must determine which module are which type.

#### No package.json
Some proposed solutions have suggested that `package.json` be used to carry this
information. The push-back here is that not all modules exist in a context where
a `package.json` file exists. For example, simple scripts such as `node ./myscript.js`,
or general command-line utilities such as `gulp` or `eslint`.

For the former, a command-line switch to Node.js could be provided, however this
would only specify the main module, and not help with modules it may depend on.

#### Tooling
Some tools in use also require knowing the module format in advance, and are not
able to parse the contents of the module or a `package.json` file in a non-trivial
manner to determine this.

### Semantic differences between module types
Once the type of module has been determined, there are some notable differences
in capabilities and semantics that must be bridged.

#### Callable modules
A CommonJS module can be any value that can be assigned to a property, and often
is a function object. (e.g. `module.exports = function()...`). An ES2015 module
cannot be any value, and can be thought of as resembling an object with getters.

This means many CommonJS modules cannot be directly represented as ES2015 modules.

#### Live bindings
Per the above, ES2015 exports can be thought of as getters. This means if importing
a value such as `import {someNumber} from "mod"`, then the value of `someNumber`
may change inbetween references in the importing module.

#### Read-only bindings
Exports from ES2015 modules are read-only (i.e. getters without setters). In CommonJS
the properties on a module are typically mutable.

#### Static bindings
Similarly to the above, in CommonJS a module is typically just as object that can
be updated at any time, including having new members added. In ES2015, a module's
exports are statically determined when the module is parsed and may not be modified.


## The solution
 
### Current proposals and issues

#### Side-by-side modules
The current proposal is that ES2015 and CommonJS modules would live side-by-side,
and the runtime would load the ES2015 version if it is capable, else would continue
to load the CommonJS version. There are a number of potential issues with this,
especially concerning maintenance and compatibility.

Regarding compatibility, this is a concern due to all the changes in semantics outlined
above between the module types. These may introduce very subtle and hard to catch bugs
before shipping, which then only appear when the Node.js runtime is upgraded - even if 
the consuming and consumed modules remain _exactly_ identical. Per the initial goals,
compatibility and reliability should be paramount, and if it becomes common that code
breaks just by moving to a new version of Node.js (one that understand ES2015 modules),
then users will be hesitant to upgrade.

Regarding maintenance, this is related to the above. If the runtime **may** load
either the CommonJS or the ES2015 version of the same module depending on the Node.js
version, then the developer has to be sure to verify both platforms. Debugging
reported issues will require using the same Node version to ensure investing the
same code as the user is executing. If a developer fixes, tests, and publishes a change, 
but forgot to update/recompile the side-by-side module, then some consumers will 
report the issue is fixed, and some will not.

A minor consideration is also the size of the packages if publishing duplicate 
versions of each module. As the graph of dependencies of typical apps grows larger,
this adds to the disk space and time required for an `npm install`.

In general, side-by-side is significantly more work and risk, and adds a significant
degree of burden to the developer to ensure both versions stay in sync and behave
identically. The reality is many developers will not get this exactly right, risking
Node.js upgrades being seen as frequently "breaking code". This is covered in more
depth in https://github.com/billti/node-es2015/blob/master/fat_packages.md .

#### Hoisting properties
The current propsoal proposes that when a CommonJS module is imported using ES2015
syntax, that the module itself is exposed as the `default` property so that it may
be callable, and the properties also "hoisted" so they may be used in an import list.

For example, if a CommonJS module "x" contains `module.exports = {"answer": 42}`, then
an ES2015 module could be written as either:

```typescript
// Use the default property
import def from "x";
console.log(def.answer);

// Use an import list
import {answer} from "x";
console.log(answer);
```

An issue with this approach is that the duplication has extra work to maintain the
ES2015 version, and also each method has different semantics (with the properties
on `default` being mutable, but the named import not). For example, to write the
ES2015 version of this, you would need to duplicate exports such as:

```typescript
export let answer = 42;
export default {answer: 42};
// Repeat duplication for every CommonJS export being modeled.
```

Note: Even this is not really identical, and has several subtle differences to the
CommonJS version - many of which a lot of developers won't be aware of.

#### New filename extension
Perhaps the most contentious issue is the proposal of a new file extension to signify
a ES2015 module (\*.mjs) versus a CommonJS module (\*.js). This solves a number of
the requirements above in a very simple and deterministic manner. However it is
also quite clearly disliked heavily by large sections of the community, and requires
editors and other tools to be updated to recognize and handle this new extension.
(As well as existing customer build scripts, gulp files, etc. to changing their
matching patterns to handle multiple extensions now).

There is also the risk and complexity that if the new extension is not adopted by
the browser ecosystem, then the current ability to share some modules between Node.js
and the browser becomes more challenging. With NPM being used more and more to
distribute JavaScript for the browser as well as for Node, this could get confusing.
(e.g. a package includes ".js" files for the CommonJS modules, more ".js" files for
the ES2015 versions for use in the browser, and ".mjs" copies for the ES2015 modules
in Node.js.

### Alternate approaches
Below are three alternate options, some of which have had variants proposed already.

These three options stand-alone, and any could be used without adopting the others.

#### No side-by-side modules
Packages should not contain both ES2015 and CommonJS versions of a module, either
of which may be loaded depending on the runtime. Per the above, this adds significant
cost and risk. If a module is going to have the CommonJS code anyway for downlevel
support, and the ES2015 version is intended to behave identically, then just always
run the CommonJS version - this guarantees the desired consistent behavior. 

Developers can still develop and debug the ES2015 source via the now
rich support for sourcemaps if desired (assuming the CommonJS was generated from the
ES2015 code). If the ES2015 version is intended to behave differently, then it should
be shipped as a different package to indicate this, (either name or version), not
demonstrate different behavior when a user updates their Node.js version.

This does mean that as long as a package/module is going to support CommonJS users,
it will ship CommonJS modules (obviously) and **only use those modules** at runtime.
If a package/module requires ES2015 specific behavior (or just only wants to ship
ES2015 code), then it will ship a package/module that **is only usable** by Node.js
versions that support ES2015 modules (and should indicate such a runtime requirement 
via the `engines` field in `package.json`).

#### No hoisting of CommonJS members
When a CommonJS module is imported using ES2015 syntax, the `module.exports` should
only be exposed as the `default` property (and not as a hoisted set of exports).

 - There is no duplication of ways to reference a member, making code consistent,
 and any ES2015 version easier to maintain.
 - The module is callable, as is common in CommonJS.
 - The module is mutable, as it is in CommonJS.
 - The binding are essentially "live", as member references are property accesses.
 - Migrating code often means just replacing `var fs = require("fs")` with `import fs from "fs"`.
(Note: Obviously the current module's exports need to have updated syntax too, but this 
is refering to module consumption).

The downside is the loss of the ability to directly import an identifier into scope. This can
easily be modeled with destructuring however if this style is really desired, e.g.

```typescript
import fs from "fs";
const {statSync, readFile} = fs;
// Note: Identifiers are no longer live bindings however
``` 

#### No new file extension
_This will be the most contentious proposal, and shouldn't preclude the above_

The challenges with using .js have been outlined above. However the challenges, cost,
and community pushback on having JavaScript modules **not** be .js files is also
significant. Any workable solution that would enable keeping the .js extension should
be seriously considered. Below is one potential solution, which includes an expansion
on (and in some cases simplification of) some ideas already discussed.

1. Packages should ideally contain just CommonJS modules, or ES2015 modules. This
would be signified in the `package.json` file via either a `main` field (for CommonJS
modules), or a `module` field (for ES2015 modules). The absense of a `package.json`
file or `module` field signifies a CommonJS module (for max compatibitly).

2. A package may support a mix of CommonJS and ES2015 modules. For this scenario,
it will require ES2015 runtime support (as there is no side-by-side) and thus require
a `package.json` file with a `module` field for the entry point. It will also contain
a `"CommonJS": [/* globs */]` field to indicate to the loader which modules are still
in CommonJS format. This allows for pre-determination of the module format without
looking at source text (or file extension).

3. Any app large enough to need to migrate gradually from CommonJS to ES2015 modules
needs a `package.json` (and almost certainly has one already for dependencies). It
would therefore use the same `package.json` fields as the above.

4. For running command-line tools in a folder (e.g. `gulp`, `grunt`, `jake`, etc.),
then again, these are likely installed as dependencies in `package.json` and the
module settings can be found there. (Note: This is not for the modules that make up
these tools, as that would be in their `package.json`, this is for how to load the
module which is that `gulpfile.js`, `jakefile.js` etc.. in your folder.

5. For running scripts directly from the command-line with no `package.json` file
present, (e.g. `node ./myscript.js`), a new switch `--module | -m` indicates the
module is an ES2015 module (e.g. `node -m ./my2015script.js`). This also allows
for piping a script via stdin (which can't use a file extension). The switch indicates
that any module imported without a `package.json` is also defaulted to ES2015 format.
(And if you are loading multiple modules with mixed formats, add a `package.json`).

The general intent of the above is that:

1. Any package you load from NPM already has a `package.json`.
2. Any project large enough to have gradual migration almost certainly has a `package.json`
3. Loose modules that you run directly probably are authored together and have a consistent format.
4. For the rare cases that don't fall into the above, you need the minor overhead of a `package.json`.
5. Most tools that need to tell the difference can be updated to understand a JSON field with globs.

Optional: For modules specified directly on the command-line (i.e. `node ./foo.js`), it is
**highly unlikely** they don't import something, as are of little use for side-effects or 
exports only. There is also minimal hit in dual-parsing the one entry point file. Thus the
above command with no switch could try parsing as CommonJS first, and if that has a parse
error (e.g. has `import` or `export` statments) try as ES2015. This will likely mean either
format works correctly practically all the time without needing the new switch.

Optional: The loader could also detect attempted assignments to a global `module.exports`
or `exports` value if loading a module as ES2015 and give a helpful message such as 
`Attempted to load CommonJS module "x" as a ES2015 module. See docs at... `

The algorithms for the above proposals are outlined in more detail on the page at
https://github.com/billti/node-es2015/blob/master/algorithms.md
