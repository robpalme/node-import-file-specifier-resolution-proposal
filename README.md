# File Specifier Resolution in Node.js

**Contributors**: Geoffrey Booth (@GeoffreyBooth), John-David Dalton (@jdalton), Jan Krems (@jkrems), Guy Bedford (@guybedford), Saleh Abdel Motaal (@SMotaal), Bradley Meck (@bmeck)

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
- This proposal only covers `import` statement specifiers; this doesn’t aim to also cover `--eval`, STDIN, command line flags, extensionless files or any of the other ways Node can import an entry point into a project or package. We intend to build on this proposal with a follow up to cover entry points.

## Real-World Data

As part of preparing the package exports proposal, @GeoffreyBooth did [research](https://gist.github.com/GeoffreyBooth/1b0d7a06bae52d124ace313634cb2f4a) into public NPM registry packages using ESM syntax already, as identified by packages that define a `"module"` field in their `package.json` files. There are 941 such packages as of 2018-10-22.

A project was created with those packages `npm install`ed, creating a gigantic `node_modules` folder containing 96,923 JavaScript (`.js` or `.mjs`) files. Code was then written to parse all of those JavaScript files with `acorn` and look for `import` or `export` declarations, and inspect the specifiers used in the `import` or `export ... from` statements. The [code for this](./esm-npm-modules-research) is in this repo. Here are the numbers:

- 5,870 `import` statements imported ESM modules (defined as NPM packages with a `"module"` field in their `package.json`) as bare specifiers, e.g. `import 'esm-module'`
- 36,712 `import` statements imported CommonJS modules (defined as packages lacking a `"module"` field) as bare specifiers, e.g. `import 'cjs-module'`
- 85,913 `import` statements imported ESM JavaScript files (defined as files with an `import` or `export` declaration), e.g. `import './esm-file.mjs'`
- 4,526 `import` statements imported CommonJS JavaScript files (defined as files with a `require` call or reference to `module.exports` or `exports` or `__filename` or `__dirname`), e.g. `import './cjs-file.js'`

## A Note on Defaults

The `--experimental-modules` implementation takes the position that `.js` files should be treated as CommonJS by default, and as of this writing there is no way to configure Node to treat them otherwise. [nodejs/modules#160](https://github.com/nodejs/modules/pull/160) contains proposals for adding a configuration block for allowing users to override this default behavior to tell Node to treat `.js` files as ESM (or more broadly, to define how Node interprets any file extension). This proposal takes the position that `.js` should be treated as ESM by default, both to follow browsers but also to be forward-looking in that ESM is the standard and should therefore be the default behavior within ESM files, rather than something to be opted into. That doesn’t mean we can’t _still_ provide such a configuration block, for example to enable the `--experimental-modules` behavior, and that might indeed be a good idea. Two proposals for configuration blocks are [`"mode"`](https://github.com/nodejs/node/pull/18392) and [`"mimes"`](https://github.com/nodejs/modules/pull/160), which are complementary to this proposal.

As `import` statements of CommonJS `.js` files appear to be far less popular than imports of ESM `.js` files (the latter are 19 times more common), we come to the conclusion that users are likely to strongly prefer `import` statements of `.js` files to treat those files as ESM rather than CommonJS as Node’s default behavior. `import` statements of `.mjs` files would always be treated as ESM, as they are in both `--experimental-modules` and the new modules implementation.

## Proposal

### `import` Statements of ESM Packages and Files

#### `import` statements of ESM packages

Importing of ESM packages is covered by the [package exports proposal](https://github.com/jkrems/proposal-pkg-exports).

#### `import` statements of ESM files

An ESM JavaScript file would use an `import` statement to import another ESM JavaScript file. Imported files must have either a `.js` or `.mjs` extension, and must be specified using a relative path or absolute URL. (Absolute paths may be supported in the future, if a system can be worked out that is compatible with browsers; see also Absolute URLs below.)

Paths must begin with `./` or `../`; specifiers that begin with an allowed package name character are treated as bare specifiers/package names and are covered by the [package exports proposal](https://github.com/jkrems/proposal-pkg-exports). Like import specifiers in browsers, Node’s import specifiers are URLs and therefore always use forward slashes and escape characters the same way URLs do.

##### Relative to the importing file: specifiers starting with `./` or `../`

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

##### Specifiers starting with `/` or `//`

These are currently unsupported but reserved for future use.

##### Absolute URLs

If the specifier parses as a valid URL, that parsed URL is used directly with no additional resolution performed. This follows [the HTML spec for resolving module specifiers](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-module-system).

```js
import config from 'file:///opt/nodejs/config.js';
```

Once Node has located the file specified by the URL, it will search up the path for a `package.json` file. If one is found, the file is treated as being inside that `package.json` package boundary and the metadata in the `package.json` governs Node’s parsing of the file (for example, if the file should be treated as ESM or as CommonJS). This prevents the possibility of two module map records for the same file where one is ESM and another is CommonJS.

### `import` Statements of CommonJS Packages and Files

Because of the problem of potentially importing the same package or identifier as both ESM or CommonJS, and therefore creating two module map entries for the same identifier, we need to be careful when we allow CommonJS to be imported into ESM files. Currently, this proposal allows importing of CommonJS only via `createRequireFromPath`; `import` statements beginning with a CommonJS package name; or `import` statements of an absolute URL where a `package.json` at or above the file being imported creates a CommonJS package boundary around the file. Other than these exceptions, `import` statements always import JavaScript as ESM. If issues are discovered with allowing `import` statements of CommonJS files (whether as deep imports or via an absolute URL), support for those cases will be dropped and users will need to use `createRequireFromPath` or any other methods we may create in the future. The priority is to enable `import` statements of CommonJS package main entry points; importing of CommonJS files inside or outside of packages will be supported whenever possible.

#### `import` statements of CommonJS package root/main entry point

An ESM JavaScript file importing a CommonJS package, where the entire import specifier is the CommonJS package name, uses the specifier resolution algorithm that CommonJS uses now to resolve the main entry point of the package. For example, where `underscore` and `pad` are both CommonJS packages:

```js
import _ from 'underscore';
// underscore has a package.json "main" field specifying "underscore.js"

import pad from 'pad';
// pad has a file named index.js in its root
```

In other words, the specifiers here behave the same as if they were in `require` calls, because the package name (`underscore` or `pad`) is detected as a CommonJS package. Packages are detected as CommonJS by _lacking_ a signifier that the package should be treated as ESM, for example the `"exports"` field from the [package exports proposal](https://github.com/jkrems/proposal-pkg-exports) or another field like [`"mode"`](https://github.com/nodejs/node/pull/18392).

“Dual-mode” packages, that export both ESM and CommonJS entry points, are loaded as ESM when imported via the `import` statement, and follow the rules of the package exports proposal. To explicitly import a dual-mode package via its CommonJS entry point, [`module.createRequireFromPath`](https://nodejs.org/docs/latest/api/modules.html#modules_module_createrequirefrompath_filename) could be used.


#### `import` statements of CommonJS files within CommonJS packages (“deep imports”)

An import specifier starting with the package name of a CommonJS package tells Node to treat any files within that package (a “deep import”) as CommonJS. Unlike `require` statements, however, file extensions are not automatically applied and `index.js` is not automatically found inside folder paths. Subpaths of CommonJS packages are always loaded as CommonJS; a subpath ending in `.mjs` would throw (for now; if it can be determined that importing `.mjs` files from within CommonJS packages can be done safely, it could be enabled in the future).

For example, where `underscore` and `aws-sdk` are both CommonJS packages:

```js
import shuffle from 'underscore/shuffle.js';
// shuffle.js is a CommonJS file at the package root

import S3 from 'aws-sdk/clients/s3.js';
// aws-sdk has a top-level folder named clients containing a CommonJS file named s3.js
```

#### `import` statements of “loose” CommonJS files (files outside of packages)

Currently, `module.createRequireFromPath` can be used to import CommonJS files that aren’t inside a CommonJS package. Seeing as there is low user demand for ESM files importing CommonJS files outside of CommonJS packages, we feel that `module.createRequireFromPath` is sufficient for now.

If user demand grows such that we want to provide a way to use `import` statements to import CommonJS files that aren’t inside CommonJS packages, we have a few options:

1. We could treat all `.js` files outside of an ESM package (detected via a `package.json` file with a signifier that the package is ESM, such as having an `"exports"` field) as CommonJS and all `.mjs` files as ESM. This would follow the current `--experimental-modules` behavior.
2. We could introduce a `.cjs` extension that Node always interprets as JavaScript with a CommonJS parse goal, the mirror of `.mjs`. (This might be a good thing to add in any case, for design symmetry.) Users could then rename their CommonJS `.js` files to use `.cjs` extensions and import them via `import` statements. We could also support symlinks, so that a `foo.cjs` symlink pointing at `foo.js` would be treated as CommonJS when imported via `import './foo.cjs';`, to support cases where users can’t rename their files for whatever reason.
3. We could implement the `"mimes"` proposal from [nodejs/modules#160](https://github.com/nodejs/modules/pull/160), which lets users control how Node treats various file extensions within a package boundary. This would let users save their ESM files with `.mjs` while keeping their CommonJS files as `.js` and use them both. This would be an opt-in to the `--experimental-modules` behavior.
4. Presumably loaders would be able to enable this functionality, deciding to treat a file as CommonJS either based on file extension or some detection inside the file source.
5. We could create some other form of configuration to enable this, like a section in `package.json` that explicitly lists files to be loaded as CommonJS.

Again, we think that user demand for this use case is so low as to not warrant supporting it any more conveniently than `module.createRequireFromPath` for now, especially since there are several other potential solutions that remain possible in the future within the design space of this proposal.

### CommonJS Files Importing ESM

CommonJS import of ESM packages or files is outside the scope of this proposal. We presume it will be enabled via `import()`, where any specifier inside `import()` is treated like an ESM `import` statement specifier. We assume that CommonJS `require` of ESM will never be natively supported.

## Prior Art

- [Package exports proposal](https://github.com/jkrems/proposal-pkg-exports)
- [`"mimes"` field proposal](https://github.com/nodejs/modules/pull/160)
- [Import Maps](https://github.com/domenic/import-maps)
- [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)
