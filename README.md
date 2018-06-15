# A Modest Proposal for ES Modules in Node.js

## Guiding Principles

- The solution must be 100% backward-compatible.
- In the future, developers should be able to write Node programs and libraries without knowledge of the CommonJS module system.
- Module resolution rules should be reasonably compatible with the module resolution rules used by browsers.
- The ability to import a legacy package is important for adoption.

## Design Summary

**1. There is no change to the behavior of `require`. It cannot be used to import ES modules.**

This ensures 100% backward-compatibility, while still allowing some freedom of design.

**2. Instead of "index.js", the entry point for ES modules is "default.js", or instead of a package.json "main", "module" is used.**

A distinct entry point ("default.js") allows us to distinguish when a user is attempting to import from a legacy package or a folder containing CommonJS modules.

**3. When `import`ing a file path, file extensions are not automatically appended.**

The default resolution algorithm used by web browsers will not automatically append file extensions.

**4. When `import`ing a directory, if a "default.js" file cannot be found, the algorithm will attempt to find an entry point using legacy `require` rules, by consulting "package.json" and looking for "index.*" files.**

This provides users with the ability to `import` from legacy packages.

**5. `import(modulePath)` asynchronously imports an ES module from CommonJS.**

This allows old-style modules to `import` from new-style modules.

**6. Node will support a `--module` flag.**

This provides the context that the module being loaded is a module, where in future this could be set by default.

## Implementation

An experimental implementation of this proposal is available at https://github.com/guybedford/node/tree/module-default, supporting NodeJS usage:

```
node --experimental-modules --module x.js
node --experimental-modules -m x.js
node --experimental-modules -m -e "export var hello = 'world'"
```

## Use Cases

### Existing modules

Since there is no change to the behavior of `require`, there is no change to the behavior of existing modules and packages.

### Supporting `import` for old-style packages

If a "default.js" file or "module" main does not exist in the package root, then it will be loaded as an old-style module with no further changes.  It just works.

### Supporting `require` for ES Module packages

Since `require` cannot be directly used to import ES modules, we need to provide an old-style "index.js" entry point if we want to allow consumers to `require` our package:

```
src/
  [ES modules]
default.js -> src/default.js
index.js
```

The purpose of the "index.js" file will be to map the ES module into an old-style module and can be as simple as:

```js
// [index.js]
module.exports = import('./src/default.js');
```

### Distributing both transpiled and native ES modules

In this usage scenario, a package is authored in ES modules and transpiled to old-style modules using a compiler like Babel.  A typical directory layout for such a project is:

```
lib/
  [Transpiled modules]
src/
  [ES modules]
index.js -> lib/index.js
```

Users that `require` the package will load the transpiled version of the code.  If we want to allow `import`ing of this package, we can add a "default.js" file.

```
lib/
  [Transpiled modules]
src/
  [ES modules]
index.js -> lib/index.js
default.js -> src/index.js
```

We might also want our transpiler to rename "default.js" source files to "index.js".

```
lib/
  [Transpiled modules]
src/
  [ES modules]
index.js -> lib/index.js
default.js -> src/default.js
```

### Gradually migrating a project to ES modules

In this scenario, a user has a large project and wants to convert old-style modules to new style modules gradually.

**Option 1: Using a transpiler**

The project uses a transpiler to convert all code to old-style modules.  Old-style modules are distributed to consumers.  When all modules have been migrated, the transpiler can be removed.

**Option 2: Replacing require sites**

When converting an old-style module to the ES module syntax, use a script to update all internal modules which reference the converted module.  The script would change occurrences of:

```js
var someModule = require('./some-module');
```

to:

```js
var someModule = (await import('./some-module.js')).default;
```

### Deep-linking into a package

A common practice with old-style packages is to allow the user to `require` individual modules within the package source:

```js
// Loads node_modules/foo/bar.js
var deepModule = require('foo/bar');
```

If the package author wants to support both `require`ing and `import`ing into a nested module, they can do so by creating a folder for each "deep link", which contains both an old-style and new-style entry point:

```
bar/
  index.js (Entry point for require)
  default.js (Entry point for import)
```

## Why "default.js"?

- "default.html" is frequently used as a folder entry point for web servers.
- The word "default" has a special, and similar meaning in ES modules.
- Despite "default" being a common English word, "default.js" is not widely used as a file name.

In a [search of all the filenames in the @latest NPM packages as of 2016-01-28](https://gist.github.com/bmeck/9b234011938cd9c1f552d41db97ad005), "default.js" was only found 23 times in a package root.  Of these packages, 8 are using "default.js" as an ES module entry point already (they are published by @zenparsing, so no surprises there).  The remaining 15 packages would need to be updated in order to allow `import`ing them from other ES modules.

As a filename, "default.js" was found 1968 times.


## Running Modules from the Command Line

When a user executes

```sh
$ node my-module.js
```

from the command line, there is absolutely no way for Node to tell whether "my-module.js" is a legacy CJS module or an ES module. Due to the need of this knowledge for various interactive scenarios such as the entry file being provided over STDIN, node will support a `--module` flag.

```sh
$ node --module my-module.js
```

## Lookup Algorithm Psuedo-Code

### LOAD_MODULE(X, Y, T)

Loads _X_ from a module at path _Y_.  _T_ is either "require" or "import".

1. If X is a core module, then
    1. return the core module
    1. STOP
1. If X begins with './' or '/' or '../'
    1. LOAD_AS_FILE(Y + X, T)
    1. LOAD_AS_DIRECTORY(Y + X, T)
1. LOAD_NODE_MODULES(X, dirname(Y), T)
1. THROW "not found"

### LOAD_AS_FILE(X, T)

1. If T is "import",
    1. If X is a file, then
        1. If extname(X) is ".js", load X as ES module text. STOP
        1. If extname(X) is ".json", parse X to a JavaScript Object.  STOP
        1. If extname(X) is ".node", load X as binary addon.  STOP
        1. THROW "not found"
1. Else,
    1. Assert: T is "require"
    1. If X is a file, load X as CJS module text.  STOP
    1. If X.js is a file, load X.js as CJS module text.  STOP
    1. If X.json is a file, parse X.json to a JavaScript Object.  STOP
    1. If X.node is a file, load X.node as binary addon.  STOP

### LOAD_AS_DIRECTORY(X, T)

1. If T is "import",
    1. If X/default.js is a file, load X/default.js as ES module text.  STOP
    1. If X/package.json is a file,
       1. Parse X/package.json, and look for "module" field.
       1. load X/(json module field) as ES module text. STOP
    1. NOTE: If neither of the above are a file, then fallback to legacy behavior
1. If X/package.json is a file,
    1. Parse X/package.json, and look for "main" field.
    1. let M = X + (json main field)
    1. LOAD_AS_FILE(M, "require")
1. If X/index.js is a file, load X/index.js as JavaScript text.  STOP
1. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
1. If X/index.node is a file, load X/index.node as binary addon.  STOP

### LOAD_NODE_MODULES(X, START, T)

1. let DIRS=NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
    1. LOAD_AS_FILE(DIR/X, T)
    1. LOAD_AS_DIRECTORY(DIR/X, T)
