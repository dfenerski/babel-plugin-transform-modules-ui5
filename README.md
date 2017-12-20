
# babel-plugin-transform-modules-ui5 for Babel 6

An unofficial Babel transformer plugin for SAP/Open UI5.

It allows you to develop SAP UI5 applications by using the latest [ES2015](http://babeljs.io/docs/learn-es2015/), including classes and modules, or even TypeScript.

## Install

```sh
npm install babel-plugin-transform-modules-ui5 --save-dev
```

or
```sh
yarn add babel-plugin-transform-modules-ui5 --dev
```

## Configure

### .babelrc

At a minimum, add `transform-modules-ui5` to the `plugins`.

```js
{
	"plugins": ["transform-modules-ui5"]
}
```

Or if you want to supply plugin options, use the array syntax.

```js
{
	"plugins": [
		["transform-modules-ui5", {
			...pluginOpts
		}]
	]
}
```


At the time of writing, the babel version is 6.26.0, which does not natively support class property syntax. To use that syntax also add the plugin `babel-plugin-syntax-class-properties`.

It is also recommended to use `babel-preset-env` to control which ES version the final code is transformed to.


## Features

There are 2 main feature categories of the plugin, and you can use both or one without the other.:

1. Converting ES modules (import/export) into sap.ui.define or sap.ui.require.
2. Converting ES classes into Control.extend(..) syntax.

This only transforms the UI5 relevant things. It does not transform everything to ES5 (for example it does not transform const/let to var). This makes it easier to use `babel-preset-env` to determine how to transform everything else.

A more detailed list includes:

+ ES2015 Imports (default, named, and dynamic)
+ ES2015 Exports (default and named)
+ Class, using inheritance and `super` keyword
	+ Static methods and fields
	+ Class properties
	+ Class property arrow functions are bound correctly in the constructor.
+ Existing `sap.ui.define` calls don't get wrapped but can still be converted.
	+ Fixes `constructor` shorthand method, if used.
+ Various options to control the class name string used.
	+ JSDoc (name, namespace, alias)
	+ Decorators (name, namespace, alias)
	+ File path based namespace, including setting a prefix.

### Converting ES modules (import/export) into sap.ui.define or sap.ui.require

The plugin will wrap any code having import/export statements in an sap.ui.define. If there is no import/export, it won't wrap.

#### Static Import

The plugin supports all of the ES6 import statements, and will convert them into sap.ui.define arguments.

```js
import Default from 'module';
import Default, { Named } from 'module';
import { Named, Named2 } from 'module';
import * as Name from 'module';
```

The plugin uses a temporary name for the initial imported variable, and then extracts the properties from it as needed.
This allows importing ES Modules which have a 'default' value, and also non-ES modules which don't.

This:

```js
import Default, { Name1, Name2 } from 'app/File'
```

Becomes:

```js
sap.ui.define(['app/file'], function(__File) {
	function _interopRequireDefault(obj) {
		return obj && obj.__esModule ? obj.default : obj;
	}
	const Default = _interopRequireDefault(__File);
	const Name1 = __File.Name1;
	const Name2 = __File.Name2;
}
```

#### Dynamic Import

ECMAScript allows for dynamic imports calls like `import(path)` that return a Promise which resolves with an ES Module.

This plugin will convert that to an async `sap.ui.require` wrapped in a Promise.
The resolved object will be a ES module or pseudo ES module having a 'default' property on it to reference the module by, to match the format used by `import()`. If the module is not a real ES module and already has a default property, the promise will be rejected with an error.

For JavaScript projects, this syntax doesn't provide much advantage over a small utility function and has the downside of not working if your module has a 'default' property. The main advantage of this syntax is with TypeScript projects, where the TypeScript compiler will know the type of the imported module, so there his no need to define a separate interface for it.


#### Export

The plugin also supports (most of) the ES modules export syntax.

```
export function f() {};
export const c = 'c';
export { a, b };
export { a as b };
export default {};
export default X;
export { X as default };
export let v; v = 'v'; // NOTE that the value here is currently not a live binding (http://2ality.com/2015/07/es6-module-exports.html)
```

Export is a bit trickier if you want your exported module to be imported by code that does not include the import inter-op. If the importing code has the inter-op logic inserted by this plugin, then you don't need to worry, and can disable the export inter-op features if desired.

Imagine a file like this:

```js
export function two() { return 2; }
export default {
	one() { return 1;}
}
```

Which might create an exported module that looks like:

```js
{
	__esModule: true,
	 default: {
			one() { return 1; }
	 },
	two() { return 2; }
}
```

The export inter-op features do their best to only return the default export rather than returning an ES module. To do this, it determines if all the named exports already exist on the default export (with the same value reference), or whether they can be added to it if there is not a naming conflict. This plugin's terminology for that is 'collapsing'.

If there is a naming conflict or other reason why the named export cannot be added to the default export, the plugin will throw an error by default.

**Note** The plugin currently doesn't support assigning named exports as properties on an anonymous statement such as an arrow function or an object literal without a variable. This will be supported in the future.

In order to determine which properties the default export already has, the plugin checks a few locations, if applicable.

+ In an object literal.

```js
export default {
	prop: val
};
// plugin knows about prop
```

+ In a variable declaration literal or assigned afterwards.

```js
const Module = {
	prop1: val
};
Module.prop2 = val2;
export default Module;
// plugin knows about prop1 and prop2
```

+ In an Object.assign(..) or _extends(..)
	+ _extends is the named used by babel and typescript when compiling object spread.
	+ This includes a recursive search for any additional objects used in the assign/extend which are defined in the upper block scope.

```js
const object1 = {
	prop1: val
};
const object2 = Object.assign({}, object1, {
	prop2: val
});
export default object2;
// plugin knows about prop1 and prop2
```

**CAUTION**: The plugin cannot check the properties on imported modules. So if they are used in Object.assign() or _extends(), the plugin will not be aware of its properties and may override them with a named export.


**Example non-solvable issues**

The following are not solvable by the plugin, and result in an error by default.

```js
export function one() {
	return 1
}

export function two() {
	return 2
}

function one_string() {
	return "one"
}

const MyUtil = {
	// The plugin can't assign these to `exports` since the definition is not just a reference to the named export.
	one: one_string,
	two: () => "two"
}
export default MyUtil;
```

##### sap.ui.define global export flag

If you need the global export flag on sap.ui.define, add `@global` to the JSDoc on the export default statement.

```js
const X = {}

/**
 * @global
 */
export default X;
```

Outputs:

```js
sap.ui.define([], function() {
	const X = {};
	return X;
}, true);
```

#### Minimal Wrapping

By default, the plugin will wrap everything in the file into the `sap.ui.define` factory function, if there is an import or an export.

However sometimes you may want to have some code run prior to the generated  `sap.ui.define` call. In that case, set the property `noWrapBeforeImport` to true and the plugin will not wrap anything before the first `import`. If there are no imports, everything will still be wrapped.

There may be a future property to minimize wrapping in the case that there are no imports (i.e. only wrap the export).

Example:

```
const X = 1;
import A from './a';
export default {
	A, X
};

//////// Generates
"use strict";
const X = 1;
sap.ui.define(["./a"], (A) => {
	return {
		A, X
	};
});
```


### Converting ES classes into Control.extend(..) syntax

By default, the plugin converts ES classes to Control.extend(..) syntax if the class extends from a class which has been imported.
So a class without a parent will not be extended.

There are a few options or some metadata you can use to control this.


#### Configuring Name or Namespace

The plugin provides a few ways to set the class name or namespace used in the `SAPClass.extend(...)` call.


##### File based namespace (default)

The default behaviour if no JSDoc or Decorator overrides are given is to use the file path to determine the namespace to prefix the class name with.

This is based on the relative path from either the babelrc `sourceRoot` property or the current working directory.

The plugin also supports supplying a namespace prefix in this mode, in case the desired namespace root is not a directory in the filesystem.

In order to pass the namespace prefix, pass it as a plugin option, and not a top-level babel option. Passing plugin options requires the array format for the plugin itself (within the outer plugins array).

```js
{
	"sourceRoot" "src/",
	"plugins": [
		["transform-modules-ui5", {
			"namespacePrefix": "my.app"
		}],
		"other-plugins"
	]
}
```

If the default file-based namespace does not work for you (perhaps the app name is not in the file hierarchy), there are a few way to override.

##### JSDoc

The simplest way to override the names is to use JSDoc. This approach will also work well with classes output from TypeScript if you configure TypeScript to generate ES6 or higher, and don't enable removeComments.

You can set the `@name`/`@alias` directly or just the `@namespace` and have the name derived from the ES6 class name.

`@name` and `@alias` behave the same; `@name` was used originally but the `@alias` JSDoc property is used in UI5 source code, so support for that was added.

```js
/**
 * @alias my.app.AController
 */
class AController extends SAPController {
	...
}

/**
 * @name my.app.AController
 */
class AController extends SAPController {
	...
}

/**
 * @namespace my.app
 */
class AController extends SAPController {
	...
}

Will all output:

const AController = SAPController.extend("my.app.AController", {
	...
});
```

##### Decorators

Alternatively, you can use decorators to override the namespace or name used. The same properties as JSDoc will work, but instead of a space, pass the string literal to the decorator function.

NOTE that using a variable is currently not supported, but will be.

```js
@alias('my.app.AController')
class AController extends SAPController {
	...
}

@name('my.app.AController')
class AController extends SAPController {
	...
}

@namespace('my.app')
class AController extends SAPController {
	...
}

const AController = SAPController.extend("my.app.AController", {
	...
});
```

### Handling metadata and renderer

Because ES6 classes are not plain objects, you can't have an object property like 'metadata'.

This plugin allows you to configure `metadata` and `renderer` as class properties (static or not) and the plugin will convert it to object properties for the extend call.

This:
```js
class MyControl extends SAPClass {
	static renderer = MyControlRenderer;
	static metadata = {
		...
	}
}
```
Becomes:
```js
const MyControl = SAPClass.extend('MyControl', {
	renderer: MyControlRenderer,
	metadata: {
		...
	}
});
```


Since class properties are an early ES proposal, TypeScript's compiler (like babel's class properties transform) moves static properties outside the class definition, and moves instance properties inside the constructor (even if TypeScript is configured to output ESNext).

To support this, the plugin will also search for static properties outside the class definition. It does not currently search in the constructor (but will in the future) so be sure to define renderer and metadata as static props if Typescript is used.


```ts
/** Typescript **/
class MyControl extends SAPClass {
	static renderer: any = MyControlRenderer;
	static metadata: any = {
		...
	};
}

/** Typescript Output **/
class MyControl extends SAPClass {}
MyControl.renderer = MyControlRenderer;
MyControl.metadata = {
	...
};

/** Final Output **/
const MyControl = SAPClass.extend('MyControl', {
	renderer: MyControlRenderer,
	metadata: {
		...
	}
});
```

**CAUTION** The plugin does not currently search for 'metadata' or 'renderer' properties inside the constructor. So don't apply Babel's class property transform plugin before this one if you have metadata/renderer as instance properties (static properties are safe).

#### Don't convert class

If you have a class which extends from an import that you don't want to convert to .extend(..) syntax, you can add the `@nonui5` (case insensitive) jsdoc or decorator to it. Also see Options below for overriding the default behaviour.

### Special Class Property Handling for Controllers

The default class property behaviour of babel is to move the property into the constructor. This plugin has a `moveControllerPropsToOnInit` option that moves them to the `onInit` function rather than the `constructor`. This is useful since the `onInit` method is called after the view's controls have been created (but not yet rendered).

When that property is enabled, any class with 'Controller' in the name or namespace, or having the JSDoc `@controller` will be treated as a controller.

This is mostly beneficial for TypeScript projects that want easy access to controls without always casting them. In typescript, the `byId(..)` method of a controller returns a `Control` instance. Rather than continually casting that to the controller type such as `sap.m.Input`, it can be useful to use a class property.

```ts
/**
 * @name app.MyController
 * @controller
 */
class MyController extends Controller {
	input: SAPInput = this.byId("input") as SAPInput;

	constructor() {
		super();
	}

	onInit(evt: sap.ui.base.Event) {
	}
}
```

Results in:
```ts
const MyController = Controller.extend("app.MyController", {
	onInit(evt) {
		this.input = this.byId("input");
	}
});
```


Of course, the alternative would be to define and instantiate the property separately. Or to cast the control whenever it's used.

```ts
/**
 * @name app.MyController
 * @controller
 */
class MyController extends Controller {
	input: SAPInput;

	onInit(evt: sap.ui.base.Event) {
		this.input = this.byId("input") as SAPInput;
	}
}
```



## Options

**Imports**
+ `noImportInteropPrefixes` (Default `['sap/']`) A list of import path prefixes which never need an import inter-opt.

**Exports**
+ `allowUnsafeMixedExports` (Default: false) No errors for unsafe mixed exports (mix of named and default export where named cannot be collapsed*)
+ `noExportCollapse` (Default: false) Skip collapsing* named exports to the default export.
+ `noExportExtend` (Default: false) Skips assigning named exports to the default export.
+ `exportAllGlobal` (Default: false) Adds the export flag to all sap.ui.define files.

**Wrapping**
+ `noWrapBeforeImport` (Default: false) Does not wrap code before the first import (if there are imports).

**Class Conversion**
+ `namespacePrefix` (Default: '') Prefix to apply to namespace derived from directory.
+ `neverConvertClass` (Default: false) Never convert classes to SAPClass.extend() syntax.
+ `onlyConvertNamedClass` (Default false) Instead of converting any class which extends from an import, only convert if there is a `@name`.
+ `moveControllerPropsToOnInit` (Default: false) Moves class props in a controller to the onInit method instead of constructor.
+ `addControllerStaticPropsToExtend` (Default: false) Moves static props of a controller to the extends call. Useful for formatters.

\* 'collapsing' named exports is a combination of simply ignoring them if their definition is the same as a property on the default export, and also assigning them to the default export.


\** This plugin also makes use of babel's standard `sourceRoot` option.

TODO more options and better description.


## Other Similar Plugins

[sergiirocks babel-plugin-transform-ui5](https://github.com/sergiirocks/babel-plugin-transform-ui5) is a great choice if you use webpack. It allows you to configure which import paths to convert to sap.ui.define syntax and leaves the rest as ES2015 import statements, which allows webpack to load them in. This plugin will have that functionality soon. Otherwise this plugin handles more edge cases with named exports, class conversion, and typescript output support.

## Example

[MagicCube's babel-plugin-ui5-example](https://github.com/MagicCube/babel-plugin-ui5-example)

My own example coming soon.

## Building with Webpack

Take a look at [ui5-loader](https://github.com/MagicCube/ui5-loader).

## Modulization / Preload

UI5 supports Modularization through a mechanism called `preload`, which can compile many JavaScript and xml files into just one preload file.

Some preload plugins:

+ Module/CLI: [openui5-preload](https://github.com/r-murphy/openui5-preload) (Mine)
+ Gulp: [gulp-ui5-lib](https://github.com/MagicCube/gulp-ui5-lib) (MagicCube)
+ Grunt: [grunt-openui5](https://github.com/SAP/grunt-openui5) (Official SAP)

## Credits

+ Thanks to MagicCube for the upstream initial work.

## TODO

+ libs support, like sergiirocks'
+ See if we can export a live binding (getter?)
+ Configuration options
	+ Support collapsing on an anonymous default export by using a temp var.
	+ Export intern control
	+ Others..

Contribute

Please do! Open an issue, or file a PR.
Issues also welcome for feature requests.

## License

MIT © 2017 Ryan Murphy
