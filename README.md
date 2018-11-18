# File Specifier Resolution in Node.js

**Contributors**: Geoffrey Booth (@GeoffreyBooth), John-David Dalton (@jdalton)

## Motivating Examples

- A project where all JavaScript is ESM.
- A project where all source is a transpiled language such as TypeScript or CoffeeScript.
- A project where some source is ESM and some is CommonJS.
- A package that aims to be imported into either a Node.js or a browser environment, without requiring a build step.

## High Level Considerations

- The baseline behavior of relative imports should match a browser’s with a simple file server. This implies that `./x` will only ever import exactly the sibling file `"x"` without appending paths or extensions. `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
- As browsers support ESM in `import` statements of `.js` files, Node.js also needs to allow ESM in `import` statements of `.js` files. (To be precise, browsers support ESM in files served via the MIME type `text/javascript`, which is the type associated with the `.js` extension and the MIME type served for `.js` files by all standard web servers.) This is covered in summary in [nodejs/modules#149](https://github.com/nodejs/modules/issues/149) with links to deeper discussions.
- Node also needs to allow ESM in `.js` files because transpiled languages such as CoffeeScript lack a way to use the file extension as a place to store metadata, the way `.mjs` does double duty both identifying the file as JavaScript and specifying an ESM parse goal. The only way CoffeeScript could do the same would be creating a new extension like `.mcoffee`, but this is impractical because of the scope of the ecosystem updates that would be required, with related packages like `gulp-coffee` and `coffee-loader` and so on needing updates. (TypeScript has similar issues, though its situation is more complex because of its type definition files.) This is covered in [nodejs/modules#150](https://github.com/nodejs/modules/pull/150).
- Along with `.js` files needing to be able to contain ESM, they also still need to be able to contain CommonJS. We need to preserve CommonJS files’ ability to `require` CommonJS `.js` files, and ESM files need some way to import `.js` CommonJS files.
- The [package exports proposal](https://github.com/jkrems/proposal-pkg-exports) covers imports of packages via bare specifiers as well as deep imports within those packages. This proposal aims to complement that one, and therefore its topics are out of scope for this proposal. File extensions (and filenames or paths) can be irrelevant for deep imports, allowing specifiers like `"lodash/forEach"` to resolve to a path like `./node_modules/lodash/collections/each.js` via the package exports map. This proposal concerns itself only with imports of files relative to the file with the `import` statement, not using bare specifiers.

## Real-World Data

As part of preparing the package exports proposal, @GeoffreyBooth did [research](https://gist.github.com/GeoffreyBooth/1b0d7a06bae52d124ace313634cb2f4a) into public NPM registry packages using ESM syntax already, as identified by packages that define a `"module"` field in their `package.json` files. There are 941 such packages as of 2018-10-22.

A project was created with those packages `npm install`ed, creating a gigantic `node_modules` folder containing 96,923 JavaScript (`.js` or `.mjs`) files. Code was then written to parse all of those JavaScript files with `acorn` and look for `import` or `export` declarations, and inspect the specifiers used in the `import` or `export ... from` statements. The [code for this](./esm-npm-modules-research) is in this repo. Here are the numbers:

- 5,870 `import` statements imported ESM modules (defined as NPM packages with a `"module"` field in their `package.json`) as bare specifiers, e.g. `import 'esm-module'`
- 36,712 `import` statements imported CommonJS modules (defined as packages lacking a `"module"` field) as bare specifiers, e.g. `import 'cjs-module'`
- 85,913 `import` statements imported ESM JavaScript files (defined as files with an `import` or `export` declaration), e.g. `import './esm-file.mjs'`
- 4,526 `import` statements imported non-ESM JavaScript files (defined as files without either an `import` or `export` statement), e.g. `import './cjs-file.js'`

## A Note on Defaults

The `--experimental-modules` implementation takes the position that `.js` files should be treated as CommonJS by default, and as of this writing there is no way to configure Node to treat them otherwise. [nodejs/modules#160](https://github.com/nodejs/modules/pull/160) contains proposals for adding a configuration block for allowing users to override this default behavior to tell Node to treat `.js` files as ESM (or more broadly, to define how Node interprets any file extension). This proposal takes the position that `.js` should be treated as ESM by default, both to follow browsers but also to be forward-looking in that ESM is the standard and should therefore be the default behavior within ESM files, rather than something to be opted into. That doesn’t mean we can’t _still_ provide such a configuration block, for example to enable the `--experimental-modules` behavior, and that might indeed be a good idea.

As `import` statements of CommonJS `.js` files appear to be far less popular than imports of ESM `.js` files (the latter are 19 times more common), we come to the conclusion that users are likely to strongly prefer `import` statements of `.js` files to treat those files as ESM rather than CommonJS as Node’s default behavior. `import` statements of `.mjs` files would always be treated as ESM, as they are in both `--experimental-modules` and the new modules implementation.

## Proposal

### ESM Files Importing ESM Files: `import` Statements File Specifiers Interface

An ESM JavaScript file would use an `import` statement to import another ESM JavaScript file. Imported files must have either a `.js` or `.mjs` extension, and must be specified using a relative path. (Absolute paths may be supported in the future, if a system can be worked out that is compatible with browsers.)

Paths must begin with `.`; specifiers that begin with a letter or `@` (or other allowed package name character) are treated as bare specifiers/package names and are covered by the [package exports proposal](https://github.com/jkrems/proposal-pkg-exports). Like import specifiers in browsers, Node’s import specifiers are URLs and therefore always use forward slashes and escape characters the same way URLs do.

#### Relative to the importing file: specifiers starting with `.`

The following `import` statements import relative files, treating them as JavaScript with an ESM parse goal:

```js
// Folder structure:
// - index.js (this file)
// - package.json
// - constants.mjs
// - helpers/temperature.mjs

import { APP_ROOT_URL } from './constants.mjs';

import { convertTemperature } from './helpers/temperature.mjs';
```

These “relative” specifiers always start with a period. A specifier such as `'constants.mjs'` (no leading period) would be treated as a bare specifier, looking for a package named `constants.mjs` rather than a sibling file with that name.

#### Specifiers starting with `/`, `//`, or `///`

These are currently unsupported but reserved for future use.

#### Absolute URLs

If the specifier is a valid URL, no additional resolution will be done and the URL will be used as-is.

### ESM Files Importing CommonJS Files Within Packages (“Deep Imports”)

An ESM JavaScript file importing a CommonJS package, or a file within a CommonJS package (a “deep import”) uses the specifier resolution algorithm that CommonJS uses now. For example, where `underscore` and `aws-sdk` are both CommonJS packages:

```js
import _ from 'underscore';
// underscore has a package.json "main" field specifying "underscore.js"

import S3 from 'aws-sdk/clients/s3';
// aws-sdk has a top-level folder named clients containing a file named s3.js
```

In other words, the specifiers here behave the same as if they were in `require` calls, because the package name (`underscore` or `aws-sdk`) is detected as a CommonJS package. (The packages are treated as CommonJS because they lack the package exports proposal’s `"exports"` field.)

“Dual-mode” packages, that export both ESM and CommonJS entry points, are loaded as ESM when imported via the `import` statement, and follow the rules of the package exports proposal. To explicitly import a dual-mode package via its CommonJS entry point, [`module.createRequireFromPath`](https://nodejs.org/docs/latest/api/modules.html#modules_module_createrequirefrompath_filename) could be used.

### ESM Files Importing “Loose” CommonJS Files (Files Outside of Packages)

Currently, `module.createRequireFromPath` can be used to import CommonJS files that aren’t inside a CommonJS package. Seeing as there is low user demand for ESM files importing CommonJS files outside of CommonJS packages, we feel that `module.createRequireFromPath` is sufficient for now.

If user demand grows such that we want to provide a way to use `import` statements to import CommonJS files that aren’t inside CommonJS packages, we have a few options:

1. We could introduce a `.cjs` extension that Node always interprets as JavaScript with a CommonJS parse goal, the mirror of `.mjs`. (This might be a good thing to add in any case, for design symmetry.) Users could then rename their CommonJS `.js` files to use `.cjs` extensions and import them via `import` statements. We could also support symlinks, so that a `foo.cjs` symlink pointing at `foo.js` would be treated as CommonJS when imported via `import './foo.cjs';`, to support cases where users can’t rename their files for whatever reason.
2. We could implement the `"mimes"` proposal from [nodejs/modules#160](https://github.com/nodejs/modules/pull/160), which lets users control how Node treats various file extensions within a package boundary. This would let users save their ESM files with `.mjs` while keeping their CommonJS files as `.js` and use them both. This would be an opt-in to the `--experimental-modules` behavior.
3. Presumably loaders would be able to enable this functionality, deciding to treat a file as CommonJS either based on file extension or some detection inside the file source.
4. We could create some other form of configuration to enable this, like a section in `package.json` that explicitly lists files to be loaded as CommonJS.

Again, we think that user demand for this use case is so low as to not warrant supporting it any more conveniently than `module.createRequireFromPath` for now, especially since there are several other potential solutions that remain possible in the future within the design space of this proposal.

### CommonJS Files Importing ESM

CommonJS import of ESM packages or files is outside the scope of this proposal. We presume it will be enabled via `import()`, where any specifier inside `import()` is treated like an ESM `import` statement specifier. We assume that CommonJS `require` of ESM will never be natively supported.

## Prior Art

- [Package exports proposal](https://github.com/jkrems/proposal-pkg-exports)
- [`"mimes"` field proposal](https://github.com/nodejs/modules/pull/160)
- [Import Maps](https://github.com/domenic/import-maps)
- [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)
