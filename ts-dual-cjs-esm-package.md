# Typescript Dual CJS/ESM Library

This document explains some how a Dual CJS/ESM SDK/package was built.

With one exception, the SDK was written in pure [Typescript](https://www.typescriptlang.org/) (TS).

## Overview
The SDK is a dual CJS/ESM npm package. [CJS (CommonJS)](https://nodejs.org/api/modules.html) allows the SDK to be used by NodeJS applicaions. [ESM (ECMAScript Module)](https://nodejs.org/api/esm.html) allows the SDK to be used by modern browser web apps. In both cases, the TS code is transpiled to JavaScript code, the primary difference between the tow being how the two module types use `import` and `requires` to reference other modules.

There are three steps required for compiling the SDK.
1. Install npm (NodeJS Package Manager),
2. Install the dependencies,
3. Build/Transpile the TS code.

On an Ubuntu based OS, install npm with

```bash
sudo apt install npm
```

## References

Some resources that were helpful in creating the SDK.

* [Building TypeScript libraries to ESM and CommonJS](https://thesametech.com/how-to-build-typescript-project/)
* [Dual packages or supporting both CJS and ESM](https://fast-check.dev/blog/2023/09/04/dual-packages-or-supporting-both-cjs-and-esm/)

### Dual Package Hazard

Beware the [Dual Package Hazard](https://nodejs.org/api/packages.html#dual-package-hazard):

> When an application is using a package that provides both CommonJS and ES module sources, there is a risk of certain bugs if both versions of the package get loaded. This potential comes from the fact that the pkgInstance created by const pkgInstance = require('pkg') is not the same as the pkgInstance created by import pkgInstance from 'pkg' (or an alternative main path like 'pkg/module').

## Typescript Code

The source layout is as folows

```bash
./src/
    index.ts        # exports methods and types needed by all consumers
    core.ts         # exports a core set of APIs
./src/util/         # Various utility functions
./src/util/eventsource/
    eventsource.js  # Local, modified, EventSource (see below)
./src/sxedl/        # SXM specific code.
```

## Typescript Transpiler

The tool used to compile the TS code is `tsc`, and the configuration settings are set in tsconfig.json. For the SDK, we have a base tsconfig.json file and three json files that extend the main one.

- tsconfig.json
  - The base config
- tsconfig.cjs.json
  - Config for emitting as a CJS module
- tsconfig.esm.json
  - Config for emitting as a ESM module
- tsconfig.types.json
  - Config for emitting Typescript types to allow a Typescript based application to use this SDK.

To do a clean build of the SDK use the `build` script defined in [package.json](#package.json). An incremental build can be done using the `compile` script.

```bash
npm run build

npm run compile
```

### Base tsconfig.json

```json
{
    "compilerOptions": {
      "target": "ES2020",         // Target JavaScript version emitted by the compiler
      "moduleResolution": "Node", // Sets the algorithm used for finding/resolving modules

      "strict": true,             // Strict TS

      "allowJs": true,            // Allow JavaScript source
      "checkJs": false,           // Do not check the JS source for issues
      "outDir": "./out",          // [optional]: prevents overwriting JS files if not extended
    },

    "include": [    // Where the source code lives
      "src/**/*"
    ],

    "paths": {      // Where to search for types and modules
      "*": ["node_modules/*"]
    }
  }
```

### CJS :: tsconfig.cjs.json

```json
{
    "extends": "./tsconfig.json",   // extend the base config
    "compilerOptions": {
        "outDir": "./lib/cjs",      // output to this directory
        "module": "commonjs"        // emit as CommonJS
    }
}
```

### ESM :: tsconfig.esm.json

```json
{
    "extends": "./tsconfig.json",   // extend the base config
    "compilerOptions": {
        "outDir": "./lib/esm",      // output to this directory
        "module": "es2020"          // emit as ESM
    }
}
```

### Types :: tsconfig.types.json

```json
{
    "extends": "./tsconfig.json",   // extend the base config
    "compilerOptions": {
        "outDir": "./lib/types",    // output to this directory
        "declaration": true,        // emit type declarations
        "emitDeclarationOnly": true // only emit type declarations
    }
}
```

## package.json

This file defines both how the package is consumed by other modules, runtime dependencies and build time dependencies and build scripts.

```json
{
  "name": "dual-cjs-esm",
  "version": "0.6.1",
  "description": "Typescript CJS/ESM Library",
  "author": "Rory McLeod",
  "license": "MID",

  "main": "./lib/cjs/index.js",         // entry point for CJS consumers
  "module": "./lib/esm/index.js",       // entry point for ESM consumers
  "types": "./lib/types/index.d.ts",    // entry point for TS based consumers

  "exports": {
    ".": {
      // Default TS types
      "types": "./lib/types/index.d.ts",
      // CJS: used when 'requires' is used to reference another module
      //   var Logger = requires('ssam').Logger; 
      "require": "./lib/cjs/index.js",
      // ESM used when 'import' is used to refererence another module
      //   import { abc } from 'ssam';
      "import": "./lib/esm/index.js",
      "default": "./lib/esm/index.js"
    },
    "./core": {
      // core exports and types
      //  import { xyz } from 'ssam/core';
      "types": "./lib/types/core.d.js",
      "require": "./lib/cjs/core.js",
      "import": "./lib/esm/core.js",
      "default": "./lib/esm/core.js"
    },
    "./package.json": "./package.json"
  },
  "typesVersions": {
    "*": {
      "core": [
        "lib/types/core.d.js"
      ]
    }
  },
  "scripts": {
    "compile": "tsc -b ./tsconfig.cjs.json ./tsconfig.esm.json ./tsconfig.types.json",
    "build": "npm run clean && npm run compile",
    "clean": "rimraf ./lib",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  },
  "keywords": [
    "CJS",
    "ESM",
    "dual"
  ],
  "browser": {      // Hints for the packager (e.g. webpack): NodeJS modules that are not required in a browser runtime environment.
    "url": false,
    "http": false,
    "https": false,
    "util": false
},
  "devDependencies": {  // build time dependencies
    "@types/eventsource": "^1.1.15",
    "@types/node": "^20.9.2",
    "@typescript-eslint/eslint-plugin": "^6.12.0",
    "@typescript-eslint/parser": "^6.12.0",
    "eslint-config-prettier": "^9.0.0",
    "eslint-plugin-prettier": "^5.0.1",
    "prettier": "^3.1.0",
    "rimraf": "^5.0.5",
    "typescript": "^5.3.2"
  },
  "dependencies": {     // runtime dependencies
    "axios": "^1.6.2"
  },
  "engines": {          // this SDK has only be tested against Node v21
    "node": ">=21.1.0"
  }
}
```

## EventSource

This package contains the source for the [EventSource](https://www.npmjs.com/package/eventsource) package. This module emulates the EventSource module that is available in the browser's runtime environment, but not in NodeJS.

The version included in this package include a fix for a [bug](https://github.com/EventSource/eventsource/issues/326) that renders it useless when running against certain servers.

The EventSource is written in JS and requires several NodeJS specific modules - url, http, https, and util. The package.json includes a 'browser' section to let the packager (webpack) know that these modules do not need to be polyfilled.
