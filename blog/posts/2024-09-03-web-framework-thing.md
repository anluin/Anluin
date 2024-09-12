# Crafting Yet Another Web Framework (1/2)

In the overcrowded arena of web frameworks, with new ones popping up more frequently than cat memes, one might ponder:

Do we genuinely need another?

Short answer: Probably not.

But hey, I have an inexplicable fondness for wheel reinvention. So here we are, tackling this well-trodden territory yet
again.

Some time ago, I dabbled in some initial experiments, and I've finally conceptualized what an ideal framework should
be – at least one that meets my persnickety standards.

## The Grand Vision

I want this project to encompass full-stack capabilities - it should seamlessly integrate both backend and frontend
operations. Creating APIs should be an effortless experience, ideally happening automatically. My goal is to call a
function from the backend in the frontend without wrestling with REST APIs.

Oh, and by the way, I'm not a huge fan of Node.js. Deno, on the other hand, I can tolerate – hence, it'll be our runtime
of choice.

## Project Anatomy

Projects leveraging this framework should have a directory structure resembling:

```plaintext
examples/hello_world/
├── backend
│   ├── deno.json
│   ├── readme.md
│   └── src
│       └── main.ts
├── frontend
│   ├── deno.json
│   ├── readme.md
│   └── src
│       └── main.ts
└── readme.md
```

## Embarking on the Adventure

```typescript
// examples/hello_world/frontend/src/main.ts

document.title = "Hello, world!";

document.body.appendChild(
    document.createTextNode("Hello World!"),
);
```

This simple `main.ts` file, when paired with a terminal incantation like `deno task serve`, should spin up a web server
delivering the following HTML:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Hello, world!</title>
    </head>
    <body>
        Hello, world!
    </body>
</html>
```

Immediate server-side rendering. Not too shabby. We might finesse this later, but JSDOM and esbuild should suffice to
transform and bundle everything into JavaScript for now.

### Kicking Things Off:

```typescript
// cli/src/main.ts
console.log("Hello, world!");
```

```jsonc
// examples/hello_world/deno.json
{
    "tasks": {
        "serve": "deno run --allow-all ../../cli/src/main.ts serve"
    }
}
```

Running `deno task serve` should produce:

```sh
$ deno task serve
Task serve deno run --allow-all ../../cli/src/main.ts
Hello, world!
```

A small victory – Deno is working!

Next up, handling the “serve” command and importing `parseArgs` from `@std/cli`:

```sh
deno add @std/cli
```

Adjusting `main.ts` in the `cli` project:

```typescript
import { parseArgs } from "@std/cli/parse-args";

function serve() {
    console.log("Hello, world!");
}

function main(args: string[]) {
    const { _: commands } = parseArgs(args, {});
    const command = commands.shift();

    switch (command) {
        case "serve":
            return serve();

        default:
            throw new Error(`unknown command: ${command}`);
    }
}

if (import.meta.main) {
    main(Deno.args);
}
```

Now, our command can trigger different functions based on the input. But let's focus on fleshing out the "serve"
function.

### Gathering Essentials:

```typescript
function serve() {
    const projectRootDirectoryPath = Deno.cwd();
    const frontendDirectoryPath = join(projectRootDirectoryPath, "frontend");
    const frontendSourceDirectoryPath = join(frontendDirectoryPath, "src");
    const frontendEntryPointFilePath = join(frontendSourceDirectoryPath, "main.ts");
}
```

Next, integrate esbuild:

```sh
deno add npm:esbuild
```

And bundle the frontend code:

```typescript
import * as esbuild from "esbuild";

// ...

const buildResult = await esbuild.build({
    write: false,
    format: "esm",
    bundle: true,
    outdir: "/",
    entryNames: "index",
    absWorkingDir: projectRootDirectoryPath,
    entryPoints: [
        frontendEntryPointFilePath,
    ],
});

const indexJsOutputFile = buildResult.outputFiles.find((outputFile) =>
    outputFile.path === "/index.js"
);

console.log(indexJsOutputFile!.text);

// ...
```

You should see something like this in the console:

```typescript
// frontend/src/main.ts
document.title = "Hello, world!";

document.body.appendChild(
  document.createTextNode("Hello World!")
);
```

Time for JSDOM to step in!

```
deno add npm:jsdom
```

```typescript
// ...
import * as jsdom from "jsdom";

// ...
const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, {});

console.log(dom.serialize());
// ...
```

Which produces:

```html
<!DOCTYPE html>
<html>
    <head></head>
    <body></body>
</html>
```

To execute our script in JSDOM, we need `vm`:

```sh
deno add npm:vm
```

We then create a `Script` object and run it within the `dom.getInternalVMContext()`:

```typescript
const indexJsOutputFile = buildResult.outputFiles.find((outputFile) =>
    outputFile.path === "/index.js"
);

const script = new vm.Script(indexJsOutputFile!.text, {
    filename: "/index.js",
});

const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, {
    runScripts: "outside-only",
});

script.runInContext(dom.getInternalVMContext());

console.log(dom.serialize());
```

Finally, HTML to send back to our eager client:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Hello, world!</title>
    </head>
    <body>Hello, world!</body>
</html>
```

Before diving into `Deno.serve`, we tweak our script to support `top-level-await`:

```typescript
const indexJsOutputFile = buildResult.outputFiles.find((outputFile) =>
    outputFile.path === "/index.js"
);

const indexJsSource = `(async () => {${indexJsOutputFile!.text}})()`;

const script = new vm.Script(indexJsSource, {
    filename: "/index.js",
});

const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, {
    runScripts: "outside-only",
});

await script.runInContext(dom.getInternalVMContext());

console.log(dom.serialize());
```

Now, let’s fire up an HTTP server with `Deno.serve`:

```typescript
// ...  
await (
    Deno.serve(async (request) => {
        return new Response(undefined, {
            status: 404,
        });
    })
        .finished
);
// ...
```

For now, all requests return a 404. Let’s validate if the request wants HTML:

```typescript
// ...  
await (
    Deno.serve(async (request) => {
        if (request.headers.get("accept")?.includes("text/html")) {
            // ...
        }

        return new Response(undefined, {
            status: 404,
        });
    })
        .finished
);
// ...
```

If so, generate and send the HTML:

```typescript
// ...
const script = new vm.Script(indexJsSource, {
    filename: "/index.js",
});

await (
    Deno.serve(async (request) => {
        if (request.headers.get("accept")?.includes("text/html")) {
            const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, {
                runScripts: "outside-only",
            });

            await script.runInContext(dom.getInternalVMContext());

            return new Response(dom.serialize(), {
                status: 200,
                headers: {
                    "Content-Type": "text/html; charset=utf-8",
                },
            });
        }

        return new Response(undefined, {
            status: 404,
        });
    })
        .finished
);
```

Voilà! Visiting `http://0.0.0.0:8000/` should now reward you with a "Hello, world!" HTML.

### The Complete Masterpiece:

```typescript
import { parseArgs } from "@std/cli/parse-args";
import { join } from "@std/path";
import { default as vm } from "vm";
import * as esbuild from "esbuild";
import * as jsdom from "jsdom";

async function serve() {
    const projectRootDirectoryPath = Deno.cwd();
    const frontendDirectoryPath = join(projectRootDirectoryPath, "frontend");
    const frontendSourceDirectoryPath = join(frontendDirectoryPath, "src");
    const frontendEntryPointFilePath = join(
        frontendSourceDirectoryPath,
        "main.ts",
    );

    const buildResult = await esbuild.build({
        write: false,
        platform: "browser",
        format: "esm",
        bundle: true,
        outdir: "/",
        entryNames: "index",
        absWorkingDir: projectRootDirectoryPath,
        entryPoints: [
            frontendEntryPointFilePath,
        ],
    });

    const indexJsOutputFile = buildResult.outputFiles.find((outputFile) =>
        outputFile.path === "/index.js"
    );

    const indexJsSource = `(async () => {${indexJsOutputFile!.text}})()`;

    const script = new vm.Script(indexJsSource, {
        filename: "/index.js",
    });

    await (
        Deno.serve(async (request) => {
            if (request.headers.get("accept")?.includes("text/html")) {
                const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, {
                    runScripts: "outside-only",
                });

                await script.runInContext(dom.getInternalVMContext());

                return new Response(dom.serialize(), {
                    status: 200,
                    headers: {
                        "Content-Type": "text/html; charset=utf-8",
                    },
                });
            }

            return new Response(undefined, {
                status: 404,
            });
        })
            .finished
    );
}

async function main(args: string[]) {
    const { _: commands } = parseArgs(args, {});
    const command = commands.shift();

    switch (command) {
        case "serve":
            return await serve();

        default:
            throw new Error(`unknown command: ${command}`);
    }
}

if (import.meta.main) {
    await main(Deno.args);
}
```

In the [next post](./2024-09-05-web-framework-thing.md), we will make sure to deliver the other files that esbuild outputs to the client as well!
