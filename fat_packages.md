# The case against "fat-packages"

## tl;rd
If a module is going to exist as a CommonJS module, then it should only exist as a CommonJS
module - there is no value but much complexity in also including an ES2015 version that 
should exhibit identical behavior, as the version used at runtime should be opaque.

If a module wishes to take advantage of features only in ES2015, then it should exist only
as an ES2015 module, and not also ship a CommonJS version that has different behavior when 
loaded by "pre-ES2015 module support" Node.js runtimes. This could cause code to break
without a single change other than upgrading the Node.js version used.

Thus if there is no value in shipping both module formats with identical behavior,
and it is dangerous to ship both module formats with different behavior, you should
never ship both module formats side-by-side, either of which may be used depending on
the runtime.

Developing applications or packages containing only one version of any one module
results in simplified authoring & consumption, consistent package behavior, smaller packages,
and a clearer migration story.

## Dual mode packages
The current proposals to add ES2015 module support to the Node.js runtime - both 
the `.mjs` extension approach the Node team is currently leaning towards, as well as 
the _"In defense of .js"_ proposal by Dave and Yehuda - propose that packages contain 
both the CommonJS version of the modules in the package, as well as the ES2015 
versions of the modules. This is to provide for interop across module types as the 
ecosystem transitions. I believe this approach is suboptimal for several reasons.

### Symmetry
Under the proposals, when a module is required from a _dual-mode_ packages,
a newer "ES2015 capable" Node.js engine would load the ES2015 format modules, and older
Node.js engines would load the CommonJS format modules. This requires that both formats
be kept in sync, else the consumer of the module, without any changes in its code,
will see different behavior from the consumed module depending on the Node.js version
being used - i.e. the _exact_ same package versions may have a __breaking change__ in
their interoperability just by moving to a newer Node.js runtime. 

If you need to ship __identically__ behaving versions of the modules, what is the
value in shiping both? If you had to ship the CommonJS versions anyway, why not _only_
ship those, and reduce the size of your package? With the many subtle (and many 
not-so-subtle) differences in semantics between CommonJS and ES2015 modules, you also
incur the risk of having to track down very nuanced bugs where the behavior differs
across module types if shipping both, but avoid this risk if just shipping one format.

If you wanted to take advantage of features only available to ES2015 format modules,
(bearing in mind you can use all other ES2015 features in CommonJS format modules),
you definitely shouldn't ship such changes side-by-side with CommonJS versions that 
can't behave identically. Ship an ES2015-only package with the different behavior.

### Migration
Another argument for the side-by-side existance of CommonJS and ES2015 modules in a
package is to allow gradual adoption, with a future envisioned where all packages
contain ES2015 modules, and CommonJS is "legacy". I'd argue that this approach
results in _holding back_ ES2015 modules, because as outlined above, if a package
contains both and they need to behave identically, there is no ability to adopt new
"ES2015 only" features in the package. If you are restricted to a "CommonJS" subset
of functionality, and need to ship the CommonJS modules anyway, and all runtimes can
use them... what is the incentive to ship ES2015 modules in Node.js packages? Whereas
if the ES2015 modules are in a separate package, they can be revved independendly and
take advantage of unique features.

Also, if packages contain both, then how can package publishers determine what
number of consumers are loading the CommonJS vs ES2015 format modules from within
their package? How are they to know when it's safe to deprecate the CommonJS modules
from within the package? The download stats for packages containing both formats are of
no help.

### Complexity
Shipping dual-mode packages adds complexity to publishing. This means you now
need to maintain two compatible versions of the modules - either by hand or by
including a transpiler into your build process - and restrict yourself to code
that can run with faithful semantics once transpiled. Authoring ES2015 versions
of CommonJS modules is also a challenge, for example with the proposed hoisting
of `module.exports` onto the `default` member, a CommonJS export is available
via two paths in ES2015 consumers, (`mod.prop`, and `mod.default.prop`). This 
duplication would need to be created and maintained in the ES2015 format module 
also, else consumer code may break (again, with no change in their code, just 
depending on the Node.js version loading the module).

### Modelling
This is a challenge that has resulted in many discussions with regards to
TypeScript and the descriptions of existing libraries in type definitions. This
applies somewhat to API documentation in general however. This is closely related
to the prior point, as it only occurs when a module may be either of two formats.

If the API descriptions for module `lib` state it exports a function called `foo`,
is that a CommonJS export or an ES2015 export? If it's a CommonJS export then
it can be consumed via the below, if it's an ES2015 export then it can't.

```javascript
import lib from "lib";
lib.foo();
```

The above makes use of the hoisting of the exports onto the `default` member,
which is preferable when consuming existing CommonJS modules, as often a function
is assigned to `module.exports`, (and an imported namespace isn't callable).

## A proposal
I propose that there be no dual-mode Node.js modules. Specifically:

 - A package/app should contain only one file for each module within it. (Note: These
 can still be mixed, with module "./a" being CommonJS, and module "./b" being ES2015),
 but the source file that represents a module should never be "it depends...". 
 - A package containing ES2015 modules should set the `engines` field in `package.json`
 to require a version of Node.js that understands ES2015 format packages (e.g. 
 `"engines": {"node": ">= 8.0.0"}`).
 - Consumers that wish to support runtimes that support only CommonJS modules _by 
 definition_ cannot depend on any ES2015 format modules, and should depend only on
 the CommonJS version of packages, (and obviously be a CommonJS package themselves).

This has the following benefits:
 - Packages don't have to maintain two sets of modules with different semantics
 that try to behave identically (via transpilation or other means).
 - Packages are half the size due to the above.
 - Identical code won't risk subtle differences in module semantics depending on
 the Node.js runtime loading its dependencies.
 - API descriptions and type definitions are concrete (as they model a specific
 version of a package/module - which is only one of CommonJS or ES2015).
 - Authors can tell on NPM the metrics and dependencies for the different versions
 of their packages (e.g. the CommonJS only, or the ES2015 versions), and make an informed
 decision on when (if ever) to stop maintaining the CommonJS version.
 - Authors are free to use the latest and greatest ES2015 features in their ES2015
 modules, without being restricted to being _compatible_ with the side-by-side CommonJS
 versions.

See the [algorithms.md](https://github.com/billti/node-es2015/blob/master/algorithms.md) 
file in the repository for how Node.js would determine which 
format a module is, and how interop between CommonJS and ES2015 modules would work.
