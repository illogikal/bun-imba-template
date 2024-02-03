# imba-bun-template

First of all clone this project to your local machine. Make sure that [Bun](https://bun.sh) is installed and availble in the folder where you have cloned the project.

Install dependencies:

```bash
bun install
```

Run the project:

```bash
bun dev
```
**Known issue**: Current version of Bun's watch function do not work well with WSL on Windows. Independent of how the watch is initiated: via code or CLI. At least on my working computer. Please share your results with me.

## How it works
I don't usually use `create` methods of different CLI tools to make bootstrap projects, bacause I like to know how things work under the hood. So let's dive into details...

Bun does everything that Imba used to do out of the box. The more such tools like Bun or Vite are established the more time Imba team will be able to spend on compiler or frontend framework. 

And the first thing we need to make Bun and Imba work togehter is to show Bun how to compile  *.imba files.

### Backend development
To develop backend with Imba using Bun the only thing that is needed is the plugin for Bun. The working code of such plugin is pretty small: 
```js
const compiler = require("./node_modules/imba/dist/compiler.cjs");

export const imbaPlugin: BunPlugin = {
  name: "imba",
  async setup(build) {
    
    // when an .imba file is imported...
    build.onLoad({ filter: /\.imba$/ }, async ({ path }) => {
      let contents = '';

      // read and compile it with the imba compiler
      const file = await Bun.file(path).text();
      const out = compiler.compile(file, {
        sourcePath: path,
        platform: 'browser'
      })

      // the file has been successfully compiled
      if (!out.errors || !out.errors.length) {
        contents = out.js;
      }
      
      // and finally return the compiled source code as "js"
      return {
        contents,
        loader: "js",
      };
    });
  }
};

plugin(imbaPlugin);

```
This plugin lets Bun deal with .imba files (compile them to JavaScript). But it will not show errors produced by the Imba compiler. 

That is why the plugin in the `preload.ts` file has much more code - more than a half of it is needed to print pretty error messages generated by the Imba compiler.

After the plugin is ready it should be preloaded before everything else (that's where the file name came from). Before running Bun preloads files that are mentioned in the `bunfig.toml`:
```bash
preload = ["./preload.ts"]
```
Well, this is enough to develop and host backend projects. Just run the command with the correct entrypoint file from CLI (same as node or imba):
```bash
bun run "./index.imba" ✔️
```
But developing a frontend project is another story...

### Frontend development

#### Bundling the code
First of all Frontend development needs files bundling for serving them to the clients. But there is a problem - the current version of Bun (1.0.25) does not support plugins when the build function is called from CLI:
```bash
bun build "./src/index.imba" --outdir './public' ❌
```
So we will need to call the `Bun.build` function from the code. And since we already could run Imba code it is not a problem - we will not get our hands dirty again by JavaScript 🤣. Here is the bare minimum code that is needed to bundle Imba files:
```imba
import {imbaPlugin} from './preload.ts'

export def compile options
	await Bun.build
		plugins: [imbaPlugin] # THIS CAN'T BE MADE VIA CLI
		entrypoints: options.entrypoints || ['./src/index.imba']
		outdir: options.outdir || './public'
		target: options.target || 'node'
		sourcemap: options.sourcemap || 'none'
		minify: options.minify || true
```
The file `compile.imba` is exactly for bundling Imba projects. There is nothing special in it - just passing the plugin when calling the build function and some other parameters (the most crucial of which is the entrypoint).

#### HTTP server
After the project is bundled it is a good idea to test how it works before deploying it to the hosting. And for that Bun has a fast built-in HTTP server. Her is the bare working minimum: 
```imba
Bun.serve
	port: '8080'
	fetch: do(req)
		const path = './public' + new URL(req.url).pathname
		const file = await Bun.file(path)
		return new Response(file)
	error: do(err) return new Response(null, { status: 404 })
```
#### Hot reload
To make frontend development a pleasure the project should be rebundled on every code change, and the browser should be informed about that to reload the updated version.

Bun has a built-in function to monitor changes in directory: 
```imba
let watcher = watch(import.meta.dir + '.src', {recursive: true}, &) do(event, filename) 
	# first we need to recompile
	# second to send connected browsers a message to reload

# it is a good practice to kill watches when the program exits
process.on "SIGINT", do
	watcher.close!
	process.exit(0)
```
To be able to send message to connected browsers (even if you are developing on localhost the project could be opened in several tabs) the fronend code should keep connection with the server. To achive that the http server in the `dev.imba` file injects the `hmr.html` in the `index.html`.

The code in the `hmr.html` tries to download `favicon.png` to know if the server is alive. This is needed to get rid of `ERR_CONNECTION_REFUSED` errors in the browser console, which bothers developers with any other approach. And that is why `favicon.png` is needed for hot reload to properly work.

Moreover to show the status of hot reload the actual favicon is swapped for green circle when hot reload is working, and red circle - otherwise. So you don't need to open console to see the status of hot reload.