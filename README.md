# tsc-esm-fix
Make TS projects compatible with [esm/mjs requirements](https://nodejs.org/api/esm.html#esm_packages)

[![CI](https://github.com/antongolub/tsc-esm-fix/workflows/CI/badge.svg)](https://github.com/antongolub/tsc-esm-fix/actions)
[![David](https://img.shields.io/david/dev/antongolub/tsc-esm-fix?label=deps)](https://david-dm.org/antongolub/tsc-esm-fix?type=dev)
[![Maintainability](https://api.codeclimate.com/v1/badges/795c6c62e875c263e2fa/maintainability)](https://codeclimate.com/github/antongolub/tsc-esm-fix/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/795c6c62e875c263e2fa/test_coverage)](https://codeclimate.com/github/antongolub/tsc-esm-fix/test_coverage)
[![npm (tag)](https://img.shields.io/npm/v/tsc-esm-fix)](https://www.npmjs.com/package/tsc-esm-fix)

- [Problem](#problem)
- [Solutions](#solutions)
- [Features](#features)
- [Getting started](#getting-started)
  - [Requirements](#requirements)
  - [Install](#install)
  - [Usage examples](#usage-examples)
  - [CLI](#cli)
  - [JS/TS API](#jsts-api)
- [Alternatives](#alternatives)
- [Contributing](#contributing)
- [References](#references)
- [License](#license)

### Problem
This workaround is aimed to bypass a pair of **tsc** and **ts-jest** issues _right here and right now_. 
* [TS/13422](https://github.com/microsoft/TypeScript/issues/13422) / [TS/16577](https://github.com/microsoft/TypeScript/issues/16577): **tsc** should add `.js` extensions for relative module paths if compiled as [`es2020/esnext`](https://www.typescriptlang.org/tsconfig/#module).
* [ts-jest/1174](https://github.com/kulshekhar/ts-jest/issues/1174): `import.meta` is not allowed.

### Solutions
1. Post-process tsc-compiled outputs everytime after the build.
2. Patch project sources once as Sindre recommends in [ESM migration guide](https://github.com/sindresorhus/meta/discussions/15)

This lib may be applied in both cases.

### Features
* Injects extensions to imports/re-exports statements.
    * `import {foo} from './foo'` → `import {foo} from './foo.js'`
    * `import {baz} from 'external/baz'` → `import {baz} from 'external/baz.js'`
    * Note, [including the file extension is only necessary for packages without an "exports" field](https://nodejs.org/api/esm.html#esm_packages). So in this case all the external refs remain as are.
    * Pays attention to index files: `import {bar} from './bar'` → `import {bar} from './bar/index.js'`
* Follows `outDir` found in **tsconfig.json**.  
* Finds and replaces `__dirname` and `__filename` refs with `import.meta`.
* Changes file extensions (applied to local deps only).
* Supports Windows-based runtimes.

## Getting started
### Requirements
Node.js `^12.20.0 || ^14.13.1 || >=16.0.0`

### Install
```shell
yarn add -D tsc-esm-fix
```

### Usage examples
```shell
tsc-esm-fix [options]

# to post-process outputs each time
tsc-esm-fix --target='target/es6'

# to patch ts sources once
tsc-esm-fix --target='src/main/ts' --ext='.js'
```

```typescript
import { fix } from 'tsc-esm-fix'
await fix({
  dirnameVar: true,
  filenameVar: true,
  ext: true
})
```

**Input**  
[code ref](https://github.com/antongolub/tsc-esm-fix/blob/master/src/test/fixtures/ts-project/src/main/ts/index.ts)
```js
import { foo } from './foo';
import './bar';

// external cjs module
import * as e1def from 'e1/a/b/c';
import * as e1root from 'e1';
const { e1 } = e1def;
const { e1: e1x } = e1root;
export { e1, e1x };

// external esm module with `main` in pkg.json
export { m1 } from 'm1';
export { m1 as m1x } from 'm1/index';

// external esm module with `exports` in pkg.json
export { e2 } from 'e2';
export { e2 as es3 } from 'e2/index';
export { e2 as es4 } from 'e2/alias';
export { e2foo } from 'e2/foo';
export { e2bar } from 'e2/bar-bundle';

export * from './foo';
export * from './baz';
export * from './q/u/x';
export const foobaz = foo + 'baz';
export { foo as foo1 } from './foo.js';

// Dir with index.js file inside: ./qux.js/index.js
export { qux } from './qux.js';

export const dirname = __dirname;
export const filename = __filename;

console.log(foobaz);
```

**Output**
```js
import { foo } from './foo.js';
import './bar.js';

import * as e1def from 'e1/a/b/c/index.js';
import * as e1root from 'e1';
const { e1 } = e1def;
const { e1: e1x } = e1root;
export { e1, e1x };

export { m1 } from 'm1';
export { m1 as m1x } from 'm1/index.js';

export { e2 } from 'e2';
export { e2 as es3 } from 'e2/index';
export { e2 as es4 } from 'e2/alias';
export { e2foo } from 'e2/foo';
export { e2bar } from 'e2/bar-bundle';

export * from './foo.js';
export * from './baz/index.js';
export * from './q/u/x/index.js';
export const foobaz = foo + 'baz';
export { foo as foo1 } from './foo.js';

export { qux } from './qux.js/index.js';

export const dirname = /file:\\\\/\\\\/(.+)\\\\/[^/]/.exec(import.meta.url)[1];
export const filename = /file:\\\\/\\\\/(.+)/.exec(import.meta.url)[1];
```

### CLI
```shell
tsc-esm-fix [opts]
```
| Option | Description | Default
|---|---|---|
|`--tsconfig`| Path to project's ts-config(s) | `tsconfig.json`
|`--src` | Entry points where the ts-source files are placed. If defined `src` option suppresses `target` |
|`--target` | tsc-compiled output directory | If not specified inherited from tsconfig.json **compilerOptions.outDir**
|`--dirnameVar` | Replace `__dirname` usages with `import.meta` | true
|`--filenameVar` | Replace `__filename` var references with `import.meta` statements | true
|`--ext` | Append extension to relative imports/re-exports | `.js`
|`--unlink` | Remove original files if ext changes | true
|`--cwd`| cwd | `process.cwd()`
|`--out`| Output dir. Defaults to `cwd`, so files would be overwritten | `process.cwd()`
|`--debug` | Prints debug notes

### JS/TS API
```ts
import { fix, IFixOptions } from 'tsc-esm-fix'

const fixOptions: IFixOptions = {
  tsconfig: 'tsconfig.build.json',
  dirnameVar: true,
  filenameVar: true,
  ext: true
}

await fix(fixOptions)
```
```typescript
export interface IFixOptions {
  cwd: string
  src?: string | string[]
  target?: string | string[]
  out?: string
  tsconfig?: string | string[]
  dirnameVar: boolean
  filenameVar: boolean
  ext: boolean | string
  unlink?: boolean,
  debug?: boolean | IFunction
}
```

## Alternatives
* https://github.com/mothepro/tsc-esm
* https://github.com/digital-loukoum/tsc-esm
* https://github.com/beenotung/fix-esm-import-path

## Contributing
Feel free to open any issues: bug reports, feature requests or questions.
You're always welcome to suggest a PR. Just fork this repo, write some code, add some tests and push your changes.
Any feedback is appreciated.

## References
* [TypeScript/issues/13422: TypeScript and script type="module"](https://github.com/microsoft/TypeScript/issues/13422)
* [TypeScript/issues/28288: Feature: disable extensionless imports](https://github.com/microsoft/TypeScript/issues/28288)
* [ts-jest/issues/1174: import.meta not allowed](https://github.com/kulshekhar/ts-jest/issues/1174)
* [stackoverflow.com/how-to-use-import-meta-when-testing-with-jest](https://stackoverflow.com/questions/64961387/how-to-use-import-meta-when-testing-with-jest)
* [Pure ESM package](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)
* [stackoverflow.com/alternative-for-dirname-in-node-when-using-the-experimental-modules-flag](https://stackoverflow.com/questions/46745014/alternative-for-dirname-in-node-when-using-the-experimental-modules-flag)
* [ecma262/#sec-imports](https://tc39.es/ecma262/#sec-imports)
* [ERR_REQUIRE_ESM](https://dev.to/antongolub/errrequireesm-4j0h)
* [Publishing Node modules with TypeScript and ES modules](https://blog.logrocket.com/publishing-node-modules-typescript-es-modules/)

## License
[MIT](./LICENSE)
