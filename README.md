# esm-webpack-plugin

Output an ESM library from your bundle. Adds `export` statements to the end of the bundle for the exported members. Ideal for consumption by Javascript environments that support the ESM spec (aka: all major modern browsers).

Currently only for webpack 4 and above.

## Install

```bash
npm i -D @purtuga/esm-webpack-plugin
``` 

## Usage

In your webpack configuration (`webpack.config.js`):

```javascript
const EsmWebpackPlugin = require("@purtuga/esm-webpack-plugin");

module.exports = {
    mode: "development",
    entry: "index.js",
    output: {
        library: "LIB",
        libraryTarget: "var"
    },
    //...
    plugins: [
        new EsmWebpackPlugin()
    ]
}
```

Note the use of `output.library` and `output.libraryTarget`, which indicates a library is being built and the bundle will expose it via a scoped variable named `LIB`.

>   __NOTE__: the value for `output.library` should NOT match the name of an exported library member.
 

## Options

Options are supported by providing an object to the Plugin constructor. Example:

```javascript
const EsmWebpackPlugin = require("@purtuga/esm-webpack-plugin");

module.exports = {
    //...
    plugins: [
        new EsmWebpackPlugin({
            /*...*/
        })
    ]
}
```

Supported options:

-   `{Function} exclude`: A callback function that will be used to determine if a given file name should be excluded from processing. By default, all files whose file extension does not end with `.js` or `.mjs` will be excluded (meaning: no ESM `export` statements will be added to the output file).
    Function callback will receive two arguments - the `fileName` that is being process and webpack's `chunk` object that contains that file name.
    
        {
            exclude(fileName, chunck) {
                // exclude if not a .js/.mjs file
                return !/\.m?js/i.test(fileName);
            }
        }


## Example

Given the above Usage example:

### Entry File: `index.js`

```javascript
import {horn} from "lib/noises"

export {bark} from "lib/noises"

export function makeHornNoise() {
    horn();
}

export default makeHornNoise;

```

### Library module `lib/noises.js`

```javascript
export function horn() {
    return "honk honk";
}

export function bark() {
    return "woof woof";
}

```

### Output Bundle

```javascript
var LIB = (/******/ (function(modules){/* webpack bundle code */}));

export const bark = LIB['bark'];
export const makeHornNoise = LIB['makeHornNoise'];
export default LIB['default'];

```


### Example of usage on the Browser

In the browser:

```html
<script type="module">
    import MyLibrary from "MyLibrary.js";
    MyLibrary(); // makeHornNoise
</script>
```

Or:

```html
<script type="module">
    import {bark, makeHornNoise} from "MyLibrary.js"
    bark();
    makeHornNoise();
</script>
```

# FAQ

## When using the generated ESM library, un-used exports are not removed from final output (not three-shaken)

This is, unfortunately, a drawback and limitation of this plugin. This plugin does not change how the code is bundled or structured by webpack and only adds `export` statements to the end of file in order to enable its use via ES6 `import`. Because of that, tree-shaking is not possible - all code is already bundled and stored using webpack's internal structure. The ability to possibly support tree-shaking can only truly be supported when webpack itself introduces support for generating ESM output targets.
  
My advice is to use the generated ESM modules at runtime when no build/bundling pipeline exists on a project and to `import` source directly (if that is coded as ESM) when a pipeline does exists.


## Uncaught SyntaxError: Identifier 'MyLibrary' has already been declared

Where `MyLibrary` is the same name as the `output.library` value in your `webpack.config.js` file. 

This occurs when your library exports a member that is named the same as the value found in the `output.library` value. It is suggested that you use an obscure value for `output.library` - one that has low probability of matching an exported member's name. 


## TypeError: chunk.entryModule.buildMeta.providedExports.reduce is not a function

Console output:
```
/home/prj/node_modules/@purtuga/esm-webpack-plugin/esm-webpack-plugin.js:45
    chunk.entryModule.buildMeta.providedExports.reduce((esm_exports, exportName) => {
                                                ^

TypeError: chunk.entryModule.buildMeta.providedExports.reduce is not a function
```

In order to create an ESM package, webpack must be able to identify your module exports. This error is likey due to the fact that it was not able to do that. You can run your build with `--bail --display-optimization-bailout` to see if the following message is output against your entry module: 
`ModuleConcatenation bailout: Module exports are unknown`

The root cause is likely due to exporting modules using the `*` syntax where different modules have an export named exactly the same. Example:

`index.js`
```javascript
export * from "mod1.js";
export * from "mod1.js"
```

Where both modules have an export name `foo`. To address this issue, try using named exports instead:

```javascript
export {foo} from "mod1.js";
export {foo as foo2} from "mod2.js"
```



# License

[MIT](LICENSE)
